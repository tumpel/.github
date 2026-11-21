# Triggers: product review and configuration UX

This doc checks [RFC: Triggers](RFC-TUM-Triggers.md) against three use cases, then designs how a person configures a trigger: MCP and the CLI first, the dashboard as a mirror of the same model. It proposes changes to the RFC. Each one that changes the API needs the owner's approval ([API](https://github.com/tumpel/control-plane/blob/main/rules/api.md) rule 4) and lands as an RFC revision.

## Part 1: product review

### The three use cases

| #   | Use case                                                          | Event          | Conditions                                                                                                | Report         |
| --- | ----------------------------------------------------------------- | -------------- | --------------------------------------------------------------------------------------------------------- | -------------- |
| U1  | Agent review gate: every push to a ready pull request is reviewed | `status`       | `repository.full_name`, `context` `agent-review`, `state` `pending`, `sender.login` `github-actions[bot]` | `agent-review` |
| U2  | Post-merge review when a given engineer merges in a work repo     | `pull_request` | `repository.full_name`, `action` `closed`, `pull_request.merged` `true`, `pull_request.merged_by.login`   | none           |
| U3  | Dispatch on a label: a label on an issue starts a worker          | `issues`       | `repository.full_name`, `action` `labeled`, `label.name` `agent:work`                                     | none           |

U1 and U2 work as the RFC stands. U3 does not (G3).

### Contradictions

1. C1: The RFC makes the config file the only write path (D3). The brief makes MCP and the API first-class, and personas already have both: `Personas_create`, `_update`, `_delete` next to `tumpel config apply`. An agent in claude.ai can create a persona today but cannot create a trigger. Proposal: add `Triggers_create`, `Triggers_update` and `Triggers_delete` on the persona pattern. Config apply stays and keeps its full replace.
2. C2: The ruling says the repository owns the gate and Tumpel only listens. D10 and D17 still have Tumpel write `error` to `agent-review`, and the `report` field plus the `session_trigger_commits` table exist only for that. The brief's trigger is source, filter, persona and prompt, with no report. Dropping `report` removes one field, one table, the commit lookup per event type, the turn-end report action and slice T2. The cost: a refused or silent review leaves the gate pending, so the pull request stays blocked and says "Waiting for the agent review". That is the safe failure. Tumpel shows the refusal on the trigger's page and as an event (G4). Proposal: drop `report` and D17. This needs a ruling (Q1).

### Gaps

1. G1: Nobody can check a filter before it goes live. A typo in a field path never matches, and apply accepts it (RFC open question 1). Proposal: one operation, `Triggers_test`, runs the adapter's own match and template fill on a pasted payload and returns each condition's verdict. It replaces the schema check the RFC considers, and the dashboard form and the CLI both use it.
2. G2: A start the adapter refuses leaves no trace when the trigger writes no commit status. A missing template field or no online host kills U2 and U3 silently. Proposal: the tenant appends a `TriggerStartRefused` event with `triggerId`, `deliveryId` and the refusal, so it reaches the dashboard, `tumpel events follow`, webhooks and Web Push through the RFC: Events paths. That means the adapter forwards its own refusals to the tenant through `triggerFired`.
3. G3: U3 needs the `issues` event, and D8 accepts only `pull_request`, `push` and `status`. A label is the common way to hand an issue to an agent, and `Sessions_startFromIssue` already starts a session from one. Proposal: accept `issues` too, which also means a manifest change on existing stages.
4. G4: The D5 filter table leads a person to the wrong field twice. On a `labeled` action the label just added is `label.name`. A condition on `pull_request.labels.name` fires again on every later label of the same pull request. The actor of every event is `sender.login`, and the table lists only per-event fields such as `pusher.name`. Proposal: add both rows to the table and use them in the examples.
5. G5: U2 in a work repo needs the stage's GitHub App installed on the employer's organisation. An organisation that refuses the App blocks U2 entirely. A per-repository webhook source, keyed by tenant like `/v1/circleci-events/{tenantId}`, fits the pluggable source. This design adds nothing for it now (Q3).
6. G6: D4 lets only an Owner create a trigger, and every session it starts runs as the Owner who applied it. A Member who wants U2 for their own merges asks an Owner. That holds for a small team. It needs a ruling if a tenant has many Members (Q4).

### Machinery without a use case

1. `report` and D17 (C2).
2. `source` holds one value. It stays, because it is the field the GitLab adapter plugs into, and it costs one column.
3. Path filters, negated conditions and a rate limit have no use case among U1 to U3. All three stay out. Superseded reviews (RFC open question 2) cost one review per push and close no gate wrongly, so they stay out too.

## Part 2: configuring a trigger

### The model

A trigger is the same record on every surface: the config file, MCP, the CLI and the dashboard.

| Field         | Example                                                         |
| ------------- | --------------------------------------------------------------- |
| `name`        | `agent-review`                                                  |
| `source`      | `GitHub`                                                        |
| `event`       | `status`                                                        |
| `conditions`  | `[{ "field": "state", "values": ["pending"] }]`                 |
| `personaName` | `reviewer`                                                      |
| `model`       | `claude-opus-5-5`                                               |
| `prompt`      | `Review the pull request {{target_url}} at commit {{sha}}. ...` |

A condition holds when the payload value at `field` equals one of its `values`. All conditions must hold. This doc assumes C2 is accepted. If the owner keeps `report`, every surface below gains one field: `--report CommitStatus --report-context agent-review` on the CLI, `report` in MCP, and a "Commit status" select in the form.

### Operations

| Operation         | MCP tool                | CLI command                     | Who                       |
| ----------------- | ----------------------- | ------------------------------- | ------------------------- |
| `Triggers_list`   | `tumpel_trigger_list`   | `tumpel trigger list`           | Member                    |
| `Triggers_get`    | `tumpel_trigger_get`    | `tumpel trigger get`            | Member                    |
| `Triggers_create` | `tumpel_trigger_create` | `tumpel trigger create`         | Owner, no turn token (D4) |
| `Triggers_update` | `tumpel_trigger_update` | `tumpel trigger update`         | Owner, no turn token      |
| `Triggers_delete` | `tumpel_trigger_delete` | `tumpel trigger delete`         | Owner, no turn token      |
| `Triggers_test`   | `tumpel_trigger_test`   | `tumpel trigger test`           | Member                    |
| `Sessions_list`   | `tumpel_session_list`   | `tumpel session list --trigger` | Member                    |

`Triggers_list`, `_get` and the `Sessions_list` filter are in the RFC. Create, update, delete and test are the proposals of C1 and G1. Create, update and delete write the tenant database and the central routes the way apply does (RFC D9), and each refusal is the one apply gives: 422 `InvalidRequest` for an event type outside D8, 404 for an unknown persona, 403 for a Member or a turn token.

### MCP tools

`tumpel_trigger_create` takes the model's fields as its arguments:

```json
{
  "name": "post-merge-review",
  "source": "GitHub",
  "event": "pull_request",
  "conditions": [
    { "field": "repository.full_name", "values": ["acme/billing"] },
    { "field": "action", "values": ["closed"] },
    { "field": "pull_request.merged", "values": ["true"] },
    { "field": "pull_request.merged_by.login", "values": ["jdoe"] }
  ],
  "personaName": "reviewer",
  "model": "claude-opus-5-5",
  "prompt": "{{pull_request.merged_by.login}} merged {{pull_request.html_url}} as {{pull_request.merge_commit_sha}}. Review the merged change and open an issue for each problem you find."
}
```

`tumpel_trigger_update` takes `triggerId` and any of the same fields, and keeps the rest. `conditions`, when present, replaces the whole list. `tumpel_trigger_delete` and `tumpel_trigger_get` take `triggerId`. `tumpel_trigger_list` takes no arguments.

`tumpel_trigger_test` takes either `triggerId` or an inline `trigger` with the create fields, plus `payload`, the webhook body as JSON. The inline form tests a trigger before it exists. It starts nothing and returns:

```json
{
  "matched": false,
  "conditions": [
    {
      "field": "action",
      "values": ["closed"],
      "actual": "closed",
      "holds": true
    },
    {
      "field": "pull_request.merged",
      "values": ["true"],
      "actual": "false",
      "holds": false
    }
  ],
  "prompt": "jdoe merged https://github.com/acme/billing/pull/88 as 4f1c...",
  "missingFields": []
}
```

`actual` is the payload's value as JSON text, or absent when the path leads nowhere. `missingFields` lists each template placeholder the payload lacks, which is the refusal a real start gets.

The server's `INSTRUCTIONS` gain one line: to react to a GitHub event, create a trigger, and test it on a sample payload first. The `tumpel_trigger_create` description says that only an Owner, outside a box, can call it.

### CLI

Every value comes from a flag or a file ([API](https://github.com/tumpel/control-plane/blob/main/rules/api.md) rule 7).

| Command                                     | Flags                                                                                                                                                                 |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tumpel trigger list`                       | `--json`                                                                                                                                                              |
| `tumpel trigger get <triggerId>`            | `--json`                                                                                                                                                              |
| `tumpel trigger create`                     | `--name`, `--source`, `--event`, `--where <field>=<value>` (repeatable), `--persona-name`, `--model`, `--prompt` or `--prompt-file`; or `--file <trigger.json>` alone |
| `tumpel trigger update <triggerId>`         | Any create flag. `--where` replaces every condition                                                                                                                   |
| `tumpel trigger delete <triggerId>`         |                                                                                                                                                                       |
| `tumpel trigger test`                       | `<triggerId>` or `--file <trigger.json>`, and `--payload-file <payload.json>`                                                                                         |
| `tumpel session list --trigger <triggerId>` | The existing list flags                                                                                                                                               |

`--where` splits at the first `=`. Two `--where` flags on the same field add a value to one condition, so they match either value. Flags on different fields add conditions, which must all hold. `--file` takes one trigger in the config file's JSON shape.

U1, the agent review gate (the repository's workflow is in the RFC appendix):

```sh
tumpel trigger create --name agent-review --source GitHub --event status \
  --where repository.full_name=acme/api \
  --where context=agent-review --where state=pending \
  --where sender.login='github-actions[bot]' \
  --persona-name reviewer --model claude-opus-5-5 --prompt-file prompts/agent-review.md
```

U2, the post-merge review in a work repo:

```sh
tumpel trigger create --name post-merge-jdoe --source GitHub --event pull_request \
  --where repository.full_name=acme/billing \
  --where action=closed --where pull_request.merged=true \
  --where pull_request.merged_by.login=jdoe \
  --persona-name reviewer --model claude-opus-5-5 --prompt-file prompts/post-merge.md
```

U3, dispatch on a label (needs G3):

```sh
tumpel trigger create --name work-on-label --source GitHub --event issues \
  --where repository.full_name=acme/api \
  --where action=labeled --where label.name=agent:work \
  --persona-name ticket-worker --model claude-opus-5-5 \
  --prompt 'Work on {{issue.html_url}}. Open a pull request that closes it.'
```

Test before it goes live, then watch its runs:

```sh
tumpel trigger test --file post-merge-jdoe.json --payload-file merged-88.json
#   action                         closed        = closed         holds
#   pull_request.merged            true          = false          FAILS
#   no match: 1 of 4 conditions fails
tumpel session list --trigger trg_7Hq2
tumpel events follow --event TriggerStartRefused
```

A sample payload comes from the webhook's recent deliveries in GitHub's settings, which list every delivery including those Tumpel dropped.

`tumpel config init` writes the tenant's triggers into the file. A later `tumpel config apply` deletes every trigger the file leaves out, also one created with `trigger create`, exactly as it does for personas. `config plan` shows that delete.

### Dashboard

The dashboard gains a Triggers screen in the nav, after Personas. It calls the operations above and nothing else.

| Screen      | Shows and does                                                                  | Operations                                                                                                       |
| ----------- | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Triggers    | Every trigger with its event, conditions in short, persona and last run; create | `Triggers_list`, `Sessions_list`                                                                                 |
| Trigger     | The trigger, its runs and its refused starts; edit, test, delete                | `Triggers_get`, `Triggers_update`, `Triggers_delete`, `Triggers_test`, `Sessions_list`, `Events_list`            |
| New trigger | The form, the filter builder and the test panel                                 | `Triggers_create`, `Triggers_test`, `Personas_list`, `HarnessCatalog_list`, `GitHubConnections_listRepositories` |

A Member sees every screen read-only: the form's fields are disabled and Create, Save and Delete are absent. Test stays enabled for everyone.

#### List

```text
Triggers                                                        [ New trigger ]
───────────────────────────────────────────────────────────────────────────────
Name              Event               Filter                       Persona       Last run
agent-review      status              acme/api · context           reviewer      Running · 3 min ago
                                      agent-review · state pending
post-merge-jdoe   pull_request        acme/billing · closed ·      reviewer      Completed · yesterday
                                      merged · by jdoe
work-on-label     issues              acme/api · labeled ·         ticket-worker Refused · 2 h ago
                                      agent:work                                 No host online
───────────────────────────────────────────────────────────────────────────────
```

Last run is the newest of the trigger's sessions and its `TriggerStartRefused` events. A refusal shows its reason in plain words under the state.

#### Create and edit

```text
New trigger
───────────────────────────────────────────────────────────────────────────────
Name            [ post-merge-jdoe                     ]
Source          [ GitHub ▾ ]
Event           [ pull_request ▾ ]   status · pull_request · push · issues

When all of these hold                                     ← filter builder
  Repository       [ acme/billing ▾ ]                                   [×]
  Action           [ closed ▾ ]                                         [×]
  Merged           [ yes ▾ ]                                            [×]
  Merged by        [ jdoe                ] [+ value]                    [×]
  [+ Add condition ▾]  Author · Who did it · Base branch · Head branch ·
                       Label · Draft · Any field…

Start a session
  Persona         [ reviewer ▾ ]
  Model           [ claude-opus-5-5 ▾ ]   models the persona's harness runs
  Prompt          ┌──────────────────────────────────────────────────────┐
                  │ {{pull_request.merged_by.login}} merged              │
                  │ {{pull_request.html_url}} as                         │
                  │ {{pull_request.merge_commit_sha}}. Review the merged │
                  │ change and open an issue for each problem you find.  │
                  └──────────────────────────────────────────────────────┘
                  [ Insert field ▾ ]

                                          [ Cancel ]  [ Test ]  [ Create ]
───────────────────────────────────────────────────────────────────────────────
```

The form edits the model's fields, one control per field.

1. Source lists `GitHub` only and stays visible, so a second source adds a value and changes no layout.
2. Event lists the event types the stage accepts (D8). Changing it resets the conditions' menu.
3. The filter builder writes `conditions`. Each named row is a preset over a field path, from the event's row of the D5 table: Repository is `repository.full_name`, Who did it is `sender.login`, Label is `label.name` on `issues` and `pull_request`. Repository picks from `GitHubConnections_listRepositories`. Any field takes a dotted path. Hovering a preset shows its path, so the form teaches the CLI and MCP shape. `[+ value]` adds a value to the same row, which matches either.
4. Insert field puts a `{{path}}` placeholder into the prompt. It lists the same presets plus the fields a prompt needs: `sha`, `target_url`, `pull_request.html_url`, `issue.html_url`.
5. Create and Save call `Triggers_create` and `Triggers_update`. A refusal shows under its field: an unknown persona under Persona, an event type outside D8 under Event.
6. Edit opens the same form filled from `Triggers_get`, with Delete at the bottom left behind a confirm that names the trigger.

#### Test against a sample event

Test opens a panel below the form and calls `Triggers_test` with the form's unsaved trigger, so a person tests before Create.

```text
Test against a sample event
───────────────────────────────────────────────────────────────────────────────
Paste a webhook payload from your repository's recent deliveries on GitHub.
┌──────────────────────────────────────────────────────────────────────────┐
│ { "action": "closed", "pull_request": { "merged": false, ...             │
└──────────────────────────────────────────────────────────────────────────┘
                                                                  [ Run test ]

No match. 1 of 4 conditions fails.
  ✓ Repository   repository.full_name           acme/billing
  ✓ Action       action                         closed
  ✗ Merged       pull_request.merged            false   wanted true
  ✓ Merged by    pull_request.merged_by.login   jdoe

Prompt the session would get
  jdoe merged https://github.com/acme/billing/pull/88 as 4f1c9e2...
───────────────────────────────────────────────────────────────────────────────
```

A row whose path leads nowhere reads "This payload has no such field", which catches a typo. A placeholder the payload lacks shows above the prompt: "A start fails: the payload has no `issue.html_url`."

#### Runs

```text
post-merge-jdoe                                              [ Edit ] [ Test ]
pull_request · acme/billing · closed · merged · by jdoe · reviewer · claude-opus-5-5
───────────────────────────────────────────────────────────────────────────────
Runs
  Started            Delivery     Session                         State
  Nov 20, 14:02      d81f…        Review of acme/billing#88       Completed
  Nov 19, 09:40      a03c…        none                            Refused: No host online
  Nov 18, 17:15      77be…        Review of acme/billing#85       Failed
───────────────────────────────────────────────────────────────────────────────
```

1. Runs merge `Sessions_list` with `triggerId` and the trigger's `TriggerStartRefused` events from `Events_list`. A session row opens the Session screen. The log keeps refusals for 7 days (RFC: Events D2).
2. The Session screen shows the origin in its header: "Started by trigger post-merge-jdoe, GitHub delivery d81f…", linked to the trigger.
3. The Sessions screen gains a Started by column (person, session or trigger) and a trigger filter in the query string, `?trigger=<triggerId>`.

## Open questions for the owner

1. Q1: Drop `report` and D17, so Tumpel only listens and a failed review leaves the gate pending (C2)?
2. Q2: Add `Triggers_create`, `_update`, `_delete` and `_test` (C1, G1), and the `TriggerStartRefused` event (G2)?
3. Q3: Accept the `issues` event for label dispatch (G3), and leave a per-repository webhook source for organisations that refuse the App until one asks (G5)?
4. Q4: Keep trigger writes Owner-only, so a Member's post-merge review runs as the Owner who created it (G6)?
