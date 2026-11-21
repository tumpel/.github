# RFC: Triggers, sessions that start from an external event

| Field   | Value                                                                                                                                                                                                                            |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Status  | Proposed                                                                                                                                                                                                                         |
| Scope   | Design only; the GitHub webhook in the Worker, the tenant config, the tenant Durable Object, the session token scope, the public API, MCP and the CLI                                                                            |
| Related | [RFC: Foreman](RFC-TUM-Foreman.md), [RFC: Events](RFC-TUM-Events.md), [RFC: Tenant API](RFC-TUM-TenantApi.md), [Tenant config](config.md), [ADR 000: tenets](https://github.com/tumpel/control-plane/blob/main/doc/adr/000-tenets.md), [EDD-006](006-end-to-end-proof.md) |

Storage follows [ADR-010](https://github.com/tumpel/control-plane/blob/main/doc/adr/010-bulk-data-out-of-the-database.md). The database holds metadata only, and bulk data lives in R2.

## Summary

A trigger starts a new session when an external event matches it. It holds a source, an event type, a list of conditions on the event's payload, a persona and a prompt template. GitHub is the first source, through the stage's existing GitHub App. Triggers are generic: a trigger names any event type the App receives and filters on any field of its webhook payload. Triggers live in the tenant config file.

The Worker's GitHub adapter does every GitHub-specific step. It matches each delivery against the tenant's trigger filters, drops a delivery no trigger matches, fills the prompt and writes commit statuses. The tenant Durable Object sees a source-neutral start request and never a GitHub payload.

The first use is a required `agent-review` commit status. The repository owns the gate: its own workflow sets `agent-review` to pending when a pull request leaves draft or gets a push. A trigger on that status event starts the reviewer. The reviewer sets `agent-review` to success or failure because its prompt tells it to. The second use is a post-merge review whenever a given engineer merges a pull request.

## Problem statement

1. The GitHub App subscribes to 13 event types (`APP_EVENTS`). The GitHub webhook handler acts on `workflow_run` and `installation` only, and answers 204 to every other delivery.
2. Only a person or a turn starts a session. `Session.startedBy` is `PersonOrigin` or `TurnOrigin` in protocol v0.21.0-rc.16, and a session start always writes `Person`.
3. RFC: Events R6, as RFC: Foreman D15 changes it, lets an event start a turn only through a subscription. Nothing lets an event start a session.
4. Every turn gets an installation token with `statuses: write` as `GITHUB_TOKEN` and `GH_TOKEN` (`AGENT_SCOPE` over `AGENT_PERMISSIONS`). Any session can set any commit status on any repository of the installation, as the App.

## Requirements

1. R1: GitHub is the first source, through the existing GitHub App. GitLab or another code host plugs in later as a second adapter. Source: owner's brief, 2026-11-20.
2. R2: A trigger names any GitHub event type the App receives and filters on any field already in the webhook payload: event and action (including merged), repository, author, pusher or merger, base and head branch, labels, the draft flag, and a commit status's context and state. Source: owner's ruling 2.
3. R3: The repository owns the `agent-review` gate. A workflow in the repository sets the status to pending. Tumpel never writes pending. Source: owner's ruling 1.
4. R4: The App subscribes only to the event types triggers use. The Worker drops a delivery that no trigger matches before it reaches the tenant. Source: owner's ruling 3.
5. R5: Tumpel has no tiers of sessions and no special reviewer session. Every session keeps `statuses: write`, and the design adds no verdict operation. Source: owner's ruling, 2026-11-20.
6. R6: GitHub-specific logic stays in the Worker adapter. Source: design review.
7. R7: The design adds the fewest new entities and reuses sessions, personas and the tenant config. Source: owner's brief.
8. R8: Tenet 3 holds: only a person creates a trigger, and only an external event fires it. Tenet 2 holds: Tumpel never reads the reviewer's transcript. Tenet 5 holds: a trigger lives in the tenant config file.

## The trigger

Decision D1: a trigger holds these fields. Every field is mandatory ([Data modeling](https://github.com/tumpel/control-plane/blob/main/rules/data-modeling.md) rule 3).

| Field         | Meaning                                                                     |
| ------------- | --------------------------------------------------------------------------- |
| `name`        | A unique name in the tenant, such as `agent-review`                         |
| `source`      | `GitHub`, the only value for now                                            |
| `event`       | The source's event type, such as `status` or `pull_request` (D5)            |
| `conditions`  | Zero or more conditions on the payload, all of which must hold (D5)         |
| `personaName` | The persona of every session it starts                                      |
| `model`       | The model of every session it starts                                        |
| `prompt`      | The prompt template (D6)                                                    |
| `report`      | `NoReport`, or `CommitStatus` with a `context` such as `agent-review` (D11) |

Decision D2: triggers live in the tenant config file, next to personas and secret sets ([Tenant config](config.md)). `TenantConfig` gains a `triggers` list, and `config init`, `plan` and `apply` cover it. Apply matches triggers by name and replaces the whole list, like every other part of the file. Each trigger records `appliedBy`, the principal whose apply created or last changed it. Every session it starts runs as that principal.

Decision D3: the config file is the only write path. The API adds two reads, `Triggers_list` and `Triggers_get`, each with a `tumpel_trigger_*` MCP tool and a `tumpel trigger` CLI command (tenet 6). `Triggers_get` returns the trigger alone. `Sessions_list` takes an optional `triggerId` filter, so a member lists a trigger's sessions through the existing session surface and its pagination.

Decision D4: only an Owner applies a config that creates, changes or deletes a trigger, because each trigger spends the tenant's hosts and model budget on events a stranger can cause. An apply that carries a turn token (RFC: Foreman D8) and touches a trigger gets 403, so no agent creates a trigger.

## Filters and templates

Decision D5: a condition is a `field` and one or more `values`. The field is a dotted path into the delivery's JSON payload, such as `pull_request.base.ref`. The condition holds when the payload value at that path equals one of the values. A path that crosses an array holds when any element matches, so `pull_request.labels.name` matches a label. A scalar compares by its JSON text, so `true` matches a boolean. A path GitHub leaves absent or null holds for no value. Every field already in the payload is a filter, with no per-field code:

| Filter                | Field                                              |
| --------------------- | -------------------------------------------------- |
| Action                | `action`                                           |
| Merged                | `action` `closed` and `pull_request.merged` `true` |
| Repository            | `repository.full_name`                             |
| Author                | `pull_request.user.login`                          |
| Pusher or merger      | `pusher.name`, `pull_request.merged_by.login`      |
| Base and head branch  | `pull_request.base.ref`, `pull_request.head.ref`   |
| Labels                | `pull_request.labels.name`                         |
| Draft                 | `pull_request.draft`                               |
| Status context, state | `context`, `state` on a `status` event             |

Path filters on the files a pull request changes are a follow-up. The payload holds no file list, so they cost a GitHub API call per delivery.

Decision D6: the prompt template takes `{{field}}` placeholders with the same dotted paths, such as `{{sha}}` or `{{pull_request.html_url}}`. The adapter replaces each with the payload value. An array fills as a comma-separated list. A placeholder whose field the payload lacks refuses the start (D10). The session clones the event's `repository.html_url` on its default branch, so the template tells the agent which commit to check out.

## The GitHub source

Decision D7: the GitHub adapter in the Worker owns every GitHub-specific step: it verifies `X-Hub-Signature-256`, finds the tenant through `tenantOfInstallation`, matches conditions, fills the prompt, reads the commit a report targets, and writes every commit status. Past the adapter, a start carries a trigger id, a delivery id, a repository URL, a prompt and, for a `CommitStatus` report, a commit. A GitLab adapter adds a webhook path per tenant, as the CircleCI binding does (`/v1/circleci-events/{tenantId}`), and the same five steps.

The adapter knows which payload field holds the commit for each event type: `sha` on `status`, `pull_request.head.sha` on `pull_request`, `after` on `push`. Apply refuses a `CommitStatus` trigger on an event type without one, with 422 `InvalidRequest`.

Decision D8: `APP_EVENTS` shrinks to `workflow_run`, which prebuilds use, plus the event types triggers accept: `pull_request`, `push` and `status`. GitHub always sends `installation`. Apply refuses a trigger on another event type with 422 `InvalidRequest`. A new event type for triggers is a manifest change. GitHub reads the manifest only when it creates the App, so an existing stage's operator changes the App's event list in its GitHub settings once.

Decision D9: the filter is routing data, so it lives in the central database ([Data modeling](https://github.com/tumpel/control-plane/blob/main/rules/data-modeling.md) rule 1): `trigger_routes (trigger_id, tenant_id, source, event)` with an index on `(tenant_id, source, event)`, and `trigger_conditions (trigger_id, field, value)`. The rest of the trigger lives in the tenant database. For each delivery, the adapter reads the routes of the installation's tenant for that event type with their conditions, and drops the delivery when none match. The tenant Durable Object never wakes for an unmatched delivery (tenet 8). Apply writes the routes after the tenant commit, as it writes host labels now. A failed route write leaves the trigger silent, and apply reports it.

## The start

Decision D10: the adapter sends each match to the tenant through one new Tenant API operation, `triggerFired`, an additive change ([RFC: Tenant API](RFC-TUM-TenantApi.md) D3). The tenant Durable Object starts the session through the path of `Sessions_start`, with the trigger's persona, model and `appliedBy`. The checks of `prepareStart` run unchanged. `Session.startedBy` gains a third model, `TriggerOrigin`, with `triggerId` and `deliveryId`.

The tenant table `session_trigger_origins (session_id, trigger_id, delivery_id)` records the origin in the transaction that inserts the session. `UNIQUE (trigger_id, delivery_id)` makes each delivery start once, so a redelivery from GitHub, which keeps its `X-GitHub-Delivery` id, starts nothing. `session_trigger_commits (session_id, repository, sha)` holds the commit of a `CommitStatus` trigger's session. `Sessions_list` filters on the origin table.

When a start is refused, for example with no host online or a missing template field, the adapter sets the trigger's `CommitStatus` context to `error` with the refusal's detail, so the pull request shows why.

Decision D11: this RFC changes RFC: Events R6 a second time. It becomes: an event starts a turn only through a subscription, and a session only through a trigger, that a person created. Tenet 3 already names an external trigger as a start a person allows.

## The agent-review gate

Decision D12: the repository's own workflow opens the gate. It runs on `pull_request_target` for `opened`, `reopened`, `ready_for_review` and `synchronize`, skips a draft, and sets `agent-review` to pending on the head commit with the pull request's URL as `target_url` (appendix). `pull_request_target` runs the workflow file of the base branch, so a pull request cannot rewrite it for its own run. The repository decides which pull requests get a review, for example by skipping forks. Tumpel never writes pending.

Decision D13: the `agent-review` trigger listens to `status` with three conditions: `context` `agent-review`, `state` `pending` and `sender.login` `github-actions[bot]`. The reviewer's verdict arrives as a `status` delivery too, and the `state` condition drops it.

Decision D14: the reviewer writes the verdict. It posts its findings as one pull request review, then sets `agent-review` to `success` or `failure` on the commit with `gh api` and the `GITHUB_TOKEN` every turn already holds. It does so because the trigger's prompt tells it to. Tumpel reads no transcript and judges no review (tenet 2).

Decision D15: the gate trusts every session. Any session holds `statuses: write`, so an implementer session can set `agent-review` on its own pull request. Tumpel adds no rule against it. Tumpel has no tiers of sessions and no special reviewer session, so enforcing such a rule takes a session class, a scoped token and a verdict operation. The gate holds a merge until a session reports a review, and the prompts decide which session does.

Decision D16: the Owner adds `agent-review` as a required status check in a branch protection rule or ruleset, and turns on auto-merge. Tumpel changes no repository setting (tenet 3). The workflow's pending status keeps the gate closed. Each push moves the head to a new commit, and the workflow sets it to pending, so the gate closes until the new review reports.

Decision D17: Tumpel closes a gate the reviewer left open. When the first turn of a `CommitStatus` trigger's session ends, `Completed` or `Failed`, the tenant Durable Object returns a report action in its answer to the turn-end call. The caller hands it to the adapter after the call returns. The adapter reads the context on the session's commit, and when it still reads `pending`, sets `error` with "The review ended without a verdict" and the session link. The tenant Durable Object never calls GitHub. A person then pushes again, or sends the session a turn that asks for the verdict.

## Comparison with the Foreman subscription

Decision D18: a trigger keeps its own record. A subscription sends turns to an existing session and matches session events inside `appendEvent`. A trigger starts a new session and matches source deliveries in the Worker, before the tenant (R4). Both come from a person, refuse a turn token on create (RFC: Foreman D8), and mark their work with an origin: `TurnOrigin` and `TriggerOrigin`. A merged record would carry a session id, a goal, crew tags, an expiry, a cursor and two failure counts, none of which a trigger uses.

## Delivery plan

Decision D19: each slice is one PR. The protocol slice releases first, and the CLI releases before T1, because the surface check reads the pinned CLI (RFC: Events D10). T1 depends on RFC: Foreman slice T1 for origins.

| Slice | Component       | Change                                                                                                                      |
| ----- | --------------- | --------------------------------------------------------------------------------------------------------------------------- |
| P1    | protocol        | `TenantConfig.triggers`, `Triggers_list` and `get`, `TriggerOrigin`, the `Sessions_list` filter, Tenant API `triggerFired`  |
| C1    | CLI             | `tumpel trigger list` and `get`, `tumpel session list --trigger`                                                            |
| T1    | control plane   | The trigger tables, config apply, the routes, the adapter's match and drop, the smaller `APP_EVENTS`, the start, the origin |
| T2    | control plane   | `error` on a refused start and on a first turn that left the gate pending (D10, D17)                                        |

Tests stay low in the stack: one unit test for condition matching across arrays and absent fields, one for template filling, and one for the unique start per delivery. The proof in EDD-006 adds one step: a push to a test pull request runs the workflow, the trigger starts the reviewer, and the reviewer's verdict sets `agent-review` to success.

Out of scope: path filters, a filter on fields outside the payload, superseding an older review, check runs and a re-run button.

## Appendix: the agent-review workflow and two triggers

The repository's agent-review workflow:

```yaml
name: agent-review
on:
  pull_request_target:
    types: [opened, reopened, ready_for_review, synchronize]
permissions:
  statuses: write
jobs:
  pending:
    if: ${{ !github.event.pull_request.draft && github.event.pull_request.head.repo.full_name == github.repository }}
    runs-on: ubuntu-latest
    steps:
      - env:
          GH_TOKEN: ${{ github.token }}
        run: >
          gh api repos/${{ github.repository }}/statuses/${{ github.event.pull_request.head.sha }}
          -f state=pending -f context=agent-review -f description="Waiting for the agent review"
          -f target_url=${{ github.event.pull_request.html_url }}
```

The `triggers` list of the tenant config:

```json
[
  {
    "name": "agent-review",
    "source": "GitHub",
    "event": "status",
    "conditions": [
      { "field": "repository.full_name", "values": ["tumpel/tumpel"] },
      { "field": "context", "values": ["agent-review"] },
      { "field": "state", "values": ["pending"] },
      { "field": "sender.login", "values": ["github-actions[bot]"] }
    ],
    "personaName": "reviewer",
    "model": "<a model the reviewer persona's harness runs>",
    "report": { "kind": "CommitStatus", "context": "agent-review" },
    "prompt": "Review the pull request {{target_url}} at commit {{sha}}. Check out {{sha}} and read the diff against the base branch. Post your findings as one review with inline comments through gh api. Then set the commit status agent-review on {{sha}} to success when nothing blocks a merge, else to failure, with a one-line description."
  },
  {
    "name": "post-merge-review",
    "source": "GitHub",
    "event": "pull_request",
    "conditions": [
      { "field": "repository.full_name", "values": ["tumpel/tumpel"] },
      { "field": "action", "values": ["closed"] },
      { "field": "pull_request.merged", "values": ["true"] },
      {
        "field": "pull_request.merged_by.login",
        "values": ["<the engineer's login>"]
      }
    ],
    "personaName": "reviewer",
    "model": "<a model the reviewer persona's harness runs>",
    "report": { "kind": "NoReport" },
    "prompt": "{{pull_request.merged_by.login}} merged {{pull_request.html_url}} as {{pull_request.merge_commit_sha}}. Review the merged change and open an issue for each problem you find."
  }
]
```

## Open questions

1. Field typos. A condition on a field GitHub never sends never matches, and apply accepts it. Apply can check each field against GitHub's published webhook schemas.
2. Superseded reviews. A push leaves the older review running on its old commit. Its verdict lands on a commit that is no longer the head, so it costs one review per push and changes no gate.
3. Runaway cost. One start per delivery bounds redeliveries. Nothing bounds starts per hour across a busy tenant.
