# RFC: Stages, one infrastructure definition instantiated per stage

| Field   | Value                                                                                                                                                 |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Status  | Draft                                                                                                                                                 |
| Scope   | Design only; the split into backend, infra and deploy repositories, `packages/control-plane/infra`, both `wrangler.jsonc` files, releases, CI runners |
| Related | [RFC: Tumpel](RFC-TUM-Tumpel.md), [ADR-001](001-control-plane-on-cloudflare.md), [Deploy](https://github.com/tumpel/deploy/blob/main/doc/deploy.md), mvhenten/tumpel#291                   |

## Summary

A stage is one complete, isolated control plane: its Workers, D1 database, R2 buckets, queue, Access application, tokens and hostnames. Tumpel runs one stage, `dev`, and its values sit in 34 places across this repository and two others.

This RFC splits the code into three repositories. The backend publishes versioned, checksummed release artifacts. The infra repository holds one OpenTofu stack and the program that rolls it out. The private tumpel/deploy holds one config file per stage and drives the Cloudflare builders with pinned artifact versions. Only tumpel/deploy holds stage values.

The stack creates every resource and custom domain in one apply and generates each wrangler config from its outputs. One command rolls out a stage and one tears it down. Dev moves onto the stack through import blocks and keeps every resource and byte of data. The public repositories move to GitHub-hosted runners.

## Problem statement

1. Adding a stage takes 6 manual steps in `doc/deploy.md`: an `env` block, an infra apply, a hand-copied `database_id`, a commit, a code deploy and a second infra apply.
2. `packages/control-plane/wrangler.jsonc` repeats 61 lines per stage, because wrangler environments inherit no bindings, vars or migrations. The dev D1 id appears twice.
3. `domains.tf` branches on `var.stage == "prod"`, and `zone_name` defaults to `tumpel.dev`. A stage on another domain needs a code change.
4. Six scripts and two workflows default the stage to `dev`. `packages/infra-worker` hard-codes `tumpel-infra-dev`. `tumpel-prebuild.yml` hard-codes `https://api.dev.tumpel.dev`.
5. The release workflow uploads to `tumpel-releases-dev` with one set of repository secrets. No second stage receives a release.
6. `prevent_destroy` on 5 resources blocks every teardown. Nothing tears down a stage.
7. Every job except auto-merge runs on the self-hosted runner `tumpel-build`, which `deploy/runner` provisions.

## Requirements

1. R1: Any `stageName`. Source: #291.
2. R2: A stage serves under the apex of a domain it controls. Source: #291.
3. R3: A stage serves under a subdomain of an existing domain. Source: #291.
4. R4: One-shot rollout and one-shot teardown of a whole stage. Source: #291.
5. R5: Three repositories: backend and infra open source, deploy private. tumpel/deploy is the only place that holds stage values. Source: #291, comment of 2026-11-18 21:49.
6. R6: Wrangler ids come from tofu outputs. Source: #291.
7. R7: Dev moves onto the stack with no downtime, no recreated resource and no data loss. Source: #291.
8. R8: An idle stage bills storage only. Source: tenet 8.
9. R9: The stage config file has a versioned TypeSpec schema. Source: tenets 4 and 5.
10. R10: Every Cloudflare token, secret and resource is a tofu resource. Source: `rules/architecture.md` 9.
11. R11: The public repositories run CI on GitHub-hosted runners. Source: #291, comment of 2026-11-18 21:52.
12. R12: No change to `.github/workflows` in this repository. Source: owner's brief.

| #291 requirement      | Decision |
| --------------------- | -------- |
| Any `stageName`       | D5       |
| Apex domain           | D7       |
| Subdomain of a domain | D8       |
| One-shot rollout      | D16      |
| One-shot teardown     | D17      |

## What differs per stage today

Appendix A lists the 34 values and where each lives. They fall into four groups:

| Group                        | Values | Where they live                                                                                                              |
| ---------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Stage inputs                 | 6      | `TUMPEL_STAGE`, `cf-env`, Workers Builds variables, `variables.tf` defaults, the `prod` branch in `domains.tf`               |
| Resource names and ids       | 11     | Hand-kept in both `wrangler.jsonc` files and `infra-worker`, derived again in `main.tf` and `domains.tf`                     |
| Deploy-time vars and secrets | 9      | Tofu outputs, carried through `infra-outputs-<stage>.json` into `wrangler deploy --var` and `secret bulk`                    |
| Outside the stack            | 8      | Repository secrets, `release-pins.json`, the GitHub App in D1, `cloudflare-builds-bootstrap`, proof fixtures, enrolled hosts |

`TUMPEL_SETTINGS`, the external origins, the Durable Object migrations and the `weur` location hint are the same on every stage.

## Three repositories

Decision D1: the code splits into three repositories. The split script `npm-scripts/split` builds them locally from this history, and nothing is published until the owner says so.

| Repository | Visibility  | Holds                                                                    | Publishes                                                  |
| ---------- | ----------- | ------------------------------------------------------------------------ | ---------------------------------------------------------- |
| Backend    | Open source | Control plane, dashboard Worker, guest image recipes, proof, docs, rules | Control-plane bundle, release manifest, image layers (D12) |
| Infra      | Open source | The tofu stack and the stage program (`deploy`, `teardown`, `proof-env`) | A versioned stack tarball (D13)                            |
| Deploy     | Private     | One `StageConfig` per stage, the builder projects, account bootstrap     | Nothing                                                    |

Decision D2: Appendix B places every top-level entry of this repository. `cloudflare-builds-bootstrap` and `tofu-bootstrap` merge into tumpel/deploy.

Decision D3: the backend and infra repositories hold no stage value. Every script loses its `dev` default and requires the stage file. `zone_name` loses its default. Proof scripts take their host, fixture repository and endpoint as arguments. The dev-build rule in `checkedPins` reads the stage's `supervisorChannel` instead of comparing the name with `dev`.

## The stage config

Decision D4: tumpel/protocol defines `StageConfig` with `stageConfigVersion: 1`. tumpel/deploy holds one file per stage, `stages/<stageName>.json`.

| Field                                   | Example               | Use                                                         |
| --------------------------------------- | --------------------- | ----------------------------------------------------------- |
| `stageName`                             | `dev`                 | Suffix of every account-wide resource name                  |
| `accountId`                             | Cloudflare account id | Account of every resource and of the state endpoint         |
| `zoneName`                              | `tumpel.dev`          | Zone that carries the custom domains                        |
| `baseDomain`                            | `dev.tumpel.dev`      | The zone apex or a subdomain of it                          |
| `approverEmails`                        | list                  | Emails the one-time PIN admits                              |
| `gitHubOrganization`                    | `tumpel`              | Owner the setup page registers the stage's GitHub App under |
| `protected`                             | `true`                | Refuses teardown and any plan that destroys a resource      |
| `supervisorChannel`                     | `Dev` or `Stable`     | Whether the release may pin a supervisor dev build          |
| `backendReleaseVersion`, `bundleSha256` | `v1.4.0`, sha256      | The backend release the stage runs                          |
| `infraReleaseVersion`, `infraSha256`    | `v0.3.0`, sha256      | The stack version that rolls it out                         |

The approver emails move from `cf-env`, the encrypted Workers Builds variable and `deploy.secrets.json` into the file. An email is configuration that reads back (tenet 6). Every secret the stage needs stays a tofu resource or an account bootstrap value.

Decision D5: any `stageName` that matches `^[a-z][a-z0-9]{1,15}$` is valid, the rule `variables.tf` enforces today. Every account-wide name carries it: `tumpel-control-plane-<stageName>`, `tumpel-sessions-<stageName>`, the Access application, the tokens and the state key. No code compares a stage name with a literal, so `dev` and `prod` are ordinary names.

## One stack

Decision D6: the infra repository ships one root module. Tofu takes one typed variable, `stage`, an `object` that mirrors the infra fields of `StageConfig` with the existing validations. The stage program validates the file against `@tumpel/schemas` and passes it as `TF_VAR_stage`. It looks up `otp_idp_id` and reads `bootstrap_api_token` from the account's secret store. A wrapper module per stage would add one root and a `moved` block per resource, and each stage already has its own state.

Decision D7: a stage whose `baseDomain` equals `zoneName` serves on the apex. The dashboard takes the apex, the API `api.<zone>`, the issuer `id.<zone>` and the redirect `app.<zone>`. This is the layout `domains.tf` reserves for `prod` today.

Decision D8: a stage whose `baseDomain` ends in `.<zoneName>` serves under that subdomain of an existing zone: the dashboard on `baseDomain` and the others on `api.`, `id.` and `app.<baseDomain>`. `dev.tumpel.dev` gives dev's hostnames today. Each custom domain brings its own certificate, so a two-label hostname serves TLS. A validation refuses any other `baseDomain`, and the `prod` branch goes.

Decision D9: tofu creates both Workers as `cloudflare_worker` resources, which hold the name and settings and carry no code. Every custom domain references its Worker, so one apply creates the domains in dependency order. The exclusion pass, the second apply and the retry of the infra build go. Wrangler keeps uploading code, and both set `workers_dev = false` and observability on.

Decision D10: the stage program writes `control-plane.json` and `dashboard.json` from `tofu output -json` and the base config inside the bundle. It fills in `name`, the D1 name and id, both bucket names, the queue and the vars `R2_BUCKET_NAME`, `RELEASES_BUCKET_NAME`, `ACCESS_ISSUER`, `ACCESS_AUD`, `ACCESS_PROOF_COMMON_NAME`, `ISSUER_HOSTNAME`, `DASHBOARD_ORIGIN`, `API_ORIGIN` and `DEPLOYED_COMMIT`. Both `wrangler.jsonc` files in the backend lose their `env` blocks and keep local values for `wrangler dev` and the tests. `checkDatabase` and the copied `database_id` go.

Decision D11: each stage keeps its own state, `tumpel-<stageName>.tfstate` in the account's `tumpel-tfstate` bucket. The S3 backend sets `use_lockfile = true`, so two runs never apply one state at once.

## Release artifacts

Decision D12: a backend tag publishes a GitHub release with `GITHUB_TOKEN` alone. It attaches the bundle (both Workers built with `wrangler deploy --dry-run --outdir`, their base configs and the migrations), the release manifest and the image layers, each with a `.sha256`. The manifest is today's `current.json`. Its supervisor and CLI come from `release-pins.json`, so a backend release fixes all three versions. No public repository holds a Cloudflare secret, and `npm run release:secrets` goes.

Decision D13: an infra tag publishes the stack and the stage program as one tarball with a `.sha256`.

Decision D14: each stage pulls its release. The stage program downloads the bundle and checks it against `bundleSha256` from git, so nobody swaps the bytes after review. It copies every layer the manifest names into `tumpel-releases-<stageName>`, skipping digests it holds, and writes `current.json` last. A `Stable` stage refuses a manifest whose supervisor is a dev build `X.Y.Z-main.N`. Dev stops being the publishing stage, so no stage depends on another stage's bucket.

## Deploy orchestration

Decision D15: tumpel/deploy declares one Workers Builds project per stage, `tumpel-<stageName>`, attached to `tumpel-control-plane-<stageName>` and building tumpel/deploy. Its build command downloads the infra tarball the stage file pins, checks `infraSha256`, and runs `deploy stages/<stageName>.json`. A push that changes a stage file deploys that stage. The repository runs little CI of its own: a schema check of the stage files.

What triggers a deploy beyond a stage file change stays open until the setup exists.

## One-shot rollout and teardown

Decision D16: `deploy <stage file>` runs, in order:

1. Validate the stage file.
2. `tofu apply`, refusing any destroying plan on a protected stage.
3. Download and check the bundle, then copy the release layers.
4. Generate both wrangler configs.
5. Apply D1 migrations and upsert the dashboard's OAuth client.
6. Deploy the dashboard, put `worker_secrets`, deploy the control plane, write `current.json`.

Infra and code share one run, so `infra-outputs-<stageName>.json`, `infra_fingerprint` and the retry error go, with the `tumpel-infra` project, the `tumpel-infra-dev` Worker and `packages/infra-worker`. A release that changes no infra pays one empty `tofu plan`. On a fresh stage, the program puts the secrets right after the first upload, in the same run.

Decision D17: `teardown <stage file>` refuses a protected stage and asks for the stage name typed back. It empties `tumpel-sessions-<stageName>` and `tumpel-releases-<stageName>`, because R2 deletes only an empty bucket. It runs `tofu destroy`, which removes both Workers, and deletes the state object. `prevent_destroy` leaves the 5 resources, because tofu takes no variable in `lifecycle`; `protected` guards them through the plan check. Teardown prints the GitHub App's settings page, where a person deletes the App. Hosts enrolled to the stage lose their control plane.

## CI runners

Decision D18: the backend and infra repositories run every job on GitHub-hosted `ubuntu-24.04` runners, which cost nothing for public repositories. Image builds already fall back to `ubuntu-24.04` when `BUILD_RUNNER` is unset, and the job installs `squashfs-tools` itself. tumpel/deploy runs its schema check on a hosted runner too. `deploy/runner` and the `tumpel-build` runner retire once no repository uses them. The split PRs measure minutes per job before and after (tenet 7), because the hosted runners start without the self-hosted runner's warm npm and layer caches.

## Cost

Decision D19: an idle stage bills R2 and D1 storage only. Neither Worker has a cron trigger, and the Durable Objects hibernate idle connections (tenet 8). Workers Paid bills per account. The largest storage line is the stage's copy of its release's image layers, and teardown removes it. Workers Builds drops from two projects to one per stage, and GitHub-hosted CI drops the self-hosted VM.

## Moving dev

Decision D20: dev keeps its state, its resource addresses and its names.

1. `stages/dev.json` sets `baseDomain: dev.tumpel.dev`, so every hostname resolves as today.
2. Import blocks adopt `tumpel-control-plane-dev` and `tumpel-dashboard-dev` into `cloudflare_worker` resources.
3. The D1 database keeps `ignore_changes = [name]`, so its name stays `tumpel`, and D10 reads the real name from the state.
4. The stage program refuses the apply unless the plan shows imports and in-place updates only.
5. Before the first deploy from a generated config, it runs `wrangler deploy --dry-run` with the old `env.dev` and with the generated file, and refuses on any difference in bindings, vars or migrations.

The script name stays `tumpel-control-plane-dev`, so every Durable Object, secret and binding stays, and a wrangler deploy switches versions at once. A person deletes `tumpel-infra-dev` and the `tumpel-infra` Builds project after the first run of `tumpel-dev`.

## Bootstrapping a fresh stage

1. Once per account, tumpel/deploy's account bootstrap mints `tumpel-tofu`, `tumpel-tfstate` and the token-minting key, and creates `tumpel-tfstate` and `tumpel-builder-secrets`.
2. Commit `stages/<stageName>.json` to tumpel/deploy with its two pinned releases.
3. Run the pinned stage program's `deploy` once from a checkout, because a Builds project attaches to a Worker that exists. It creates every resource and serves every hostname.
4. Add `tumpel-<stageName>` to the builder projects, so later stage file changes deploy.
5. Open `https://api.<baseDomain>/github/app/setup?organization=<gitHubOrganization>` and register the App. GitHub's manifest flow needs a person to confirm.
6. Sign in with `tumpel login --endpoint https://api.<baseDomain>`, mint an enrolment token and enrol a host.
7. Run the backend's end-to-end proof against the stage.

## Delivery

Decision D21: each slice is one PR and updates the deploy docs for what it changes. S1 to S4 land here before the split, so the split moves finished files. Until tumpel/deploy exists, `stages/dev.json` sits in this repository, and X1 moves it there.

| Slice | Repository      | Change                                                                                                                                |
| ----- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| P1    | tumpel/protocol | `StageConfig`                                                                                                                         |
| S1    | tumpel          | The `stage` variable, D5 to D9, dev's imports, one apply, `use_lockfile`, `protected` replaces `prevent_destroy`, no `dev` defaults   |
| S2    | tumpel          | Generated wrangler configs, both `env` blocks and `checkDatabase` deleted, the dry-run comparison                                     |
| S3    | tumpel          | Bundle and manifest as release assets, pulled releases, one deploy command, `infra-worker`, the handoff and `release:secrets` deleted |
| S4    | tumpel          | `teardown`, proven on a throwaway stage                                                                                               |
| X1    | tumpel          | `npm-scripts/split`, which builds the three repositories locally per Appendix B                                                       |
| D1    | tumpel/deploy   | `stages/dev.json`, the builder project per stage, account bootstrap, the schema check                                                 |
| X2    | backend, infra  | GitHub-hosted workflows, with minutes per job before and after                                                                        |

S3 and X2 change workflows and need the owner's permission.

## Open questions

1. GitHub caps a release asset at 2 GiB, and the image layers carry harness binaries under their vendors' licences. Slice S3 must confirm that every layer fits and may ship in a public release.
2. Slice S1 must confirm that a `cloudflare_worker` adopts a Worker wrangler created, and that a later `wrangler deploy` leaves it without drift.
3. Slice S4 must show that destroying a Worker deletes its Durable Objects' storage.
4. What triggers a deploy beyond a stage file change.

## FAQ

1. Where does a stage value live? In `stages/<stageName>.json` in tumpel/deploy, nowhere else (D3).
2. Can a stage use its own domain? Yes, on the apex (D7) or under a subdomain of an existing zone (D8).
3. Does dev change hostname? No. `baseDomain: dev.tumpel.dev` yields the same four hostnames (D8).
4. Can dev lose data during the move? The plan check refuses anything but imports and in-place updates, and the Worker keeps its name (D20).
5. Why one bundle instead of building per stage? Every stage runs the bytes dev proved, checked by sha256 in git (D14).
6. Does a public repository hold a Cloudflare secret? No. Releases publish with `GITHUB_TOKEN`, and stages pull (D12).
7. Who registers the GitHub App? A person, once per stage, on its setup page.
8. What does an idle stage cost? Its R2 and D1 storage (D19).
9. How is prod protected from teardown? `protected: true` in its stage file (D17).

## Appendix A: values that differ per stage

| #   | Value                                       | Where it lives today                                                                                                                                                                                                                                                                                                 |
| --- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Stage name                                  | `TUMPEL_STAGE` default `dev` in `deploy.mjs`, `release-secrets.mjs`, `proof-env.mjs`, `e2e-proof.mjs`, `release-download.mjs`, `release-publish.mjs`; `inputs.stage \|\| 'dev'` in `release.yml` and `images.yml`; Builds variable `TUMPEL_STAGE`; tofu `var.stage`, embedded in every Access, token and bucket name |
| 2   | Cloudflare account id                       | `cf-env`, Builds variable `TUMPEL_ACCOUNT_ID`, repository secret `CLOUDFLARE_ACCOUNT_ID`, `TF_VAR_account_id`, the state endpoint                                                                                                                                                                                    |
| 3   | Zone                                        | `zone_name` default `tumpel.dev` in `variables.tf`                                                                                                                                                                                                                                                                   |
| 4   | Base domain layout                          | `var.stage == "prod"` branch in `domains.tf` locals                                                                                                                                                                                                                                                                  |
| 5   | Approver emails                             | `cf-env`, Builds variable `TUMPEL_APPROVER_EMAILS`, `deploy.secrets.json`, `var.approver_emails`                                                                                                                                                                                                                     |
| 6   | One-time PIN login method id                | Looked up per account by `deploy.mjs`, `var.otp_idp_id`                                                                                                                                                                                                                                                              |
| 7   | API hostname `api.dev.tumpel.dev`           | `domains.tf`; handoff `hostname`; `apiOrigin` in `tumpel-prebuild.yml`; install commands in `doc/deploy.md`                                                                                                                                                                                                          |
| 8   | Dashboard hostname `dev.tumpel.dev`         | `domains.tf`, handoff `dashboard_hostname`                                                                                                                                                                                                                                                                           |
| 9   | Redirect hostname `app.dev.tumpel.dev`      | `domains.tf`                                                                                                                                                                                                                                                                                                         |
| 10  | Issuer hostname `id.dev.tumpel.dev`         | `domains.tf`, handoff `issuer_hostname`                                                                                                                                                                                                                                                                              |
| 11  | Control plane Worker name                   | `name` plus `--env` in `control-plane/wrangler.jsonc`; `service` in `domains.tf`; `workerExists` in `deploy.mjs`                                                                                                                                                                                                     |
| 12  | Dashboard Worker name                       | `env.dev` and `env.prod` in `dashboard-site/wrangler.jsonc`; `service` in `domains.tf`                                                                                                                                                                                                                               |
| 13  | Infra Worker name `tumpel-infra-dev`        | `infra-worker/wrangler.jsonc`                                                                                                                                                                                                                                                                                        |
| 14  | D1 database id                              | Top level and `env.dev` of `control-plane/wrangler.jsonc`, copied by hand; `checkDatabase` in `deploy.mjs`                                                                                                                                                                                                           |
| 15  | D1 database name `tumpel`                   | `control-plane/wrangler.jsonc`; `ignore_changes` in `main.tf`                                                                                                                                                                                                                                                        |
| 16  | Sessions bucket                             | Binding and var `R2_BUCKET_NAME` in `wrangler.jsonc`; `main.tf`                                                                                                                                                                                                                                                      |
| 17  | Releases bucket                             | Binding and var `RELEASES_BUCKET_NAME` in `wrangler.jsonc`; `main.tf`; `release-publish.mjs`                                                                                                                                                                                                                         |
| 18  | Releases queue                              | Consumer in `wrangler.jsonc`; `main.tf`                                                                                                                                                                                                                                                                              |
| 19  | Tofu state key and bucket                   | `deploy.mjs`, `release-secrets.mjs`, `proof-env.mjs`; bucket in `versions.tf`                                                                                                                                                                                                                                        |
| 20  | Handoff object `infra-outputs-<stage>.json` | `deploy.mjs`, in `tumpel-builder-secrets`                                                                                                                                                                                                                                                                            |
| 21  | `ACCESS_ISSUER`                             | Tofu output, handoff, `wrangler deploy --var`                                                                                                                                                                                                                                                                        |
| 22  | `ACCESS_AUD`                                | Tofu output, handoff, `wrangler deploy --var`                                                                                                                                                                                                                                                                        |
| 23  | `ACCESS_PROOF_COMMON_NAME`                  | Tofu output, handoff, `wrangler deploy --var`                                                                                                                                                                                                                                                                        |
| 24  | `ISSUER_HOSTNAME`                           | Handoff, `wrangler deploy --var`                                                                                                                                                                                                                                                                                     |
| 25  | `API_ORIGIN` and `DASHBOARD_ORIGIN`         | Handoff, `wrangler deploy --var` on both Workers                                                                                                                                                                                                                                                                     |
| 26  | Worker secrets (6 keys)                     | Tofu output `worker_secrets`, `wrangler secret bulk`                                                                                                                                                                                                                                                                 |
| 27  | Dashboard OAuth redirect URI                | D1 upsert in `deploy.mjs`                                                                                                                                                                                                                                                                                            |
| 28  | Proof service token                         | `proof.tf`, read by `proof-env.mjs`                                                                                                                                                                                                                                                                                  |
| 29  | Release upload keys                         | One set of repository secrets, written by `release-secrets.mjs` for `mvhenten/tumpel`                                                                                                                                                                                                                                |
| 30  | Release pins and the dev-build rule         | `release-pins.json`; `stage !== "dev"` in `release-download.mjs`                                                                                                                                                                                                                                                     |
| 31  | GitHub App                                  | D1 `github_apps` through `/github/app/setup`; owner `tumpel` in `doc/deploy.md`                                                                                                                                                                                                                                      |
| 32  | Workers Builds projects                     | `tumpel` and `tumpel-infra` in `cloudflare-builds-bootstrap`, with branch, watch paths and build variables                                                                                                                                                                                                           |
| 33  | Proof fixtures                              | `TUMPEL_PROOF_HOST` default `tumpel-host`, `TUMPEL_PROOF_PREBUILD_REPO` `tumpel/prebuild-fixture`                                                                                                                                                                                                                    |
| 34  | Enrolled hosts                              | Each host's supervisor enrols against one stage's API origin with an enrolment token from that stage's D1                                                                                                                                                                                                            |

After this RFC, rows 1 to 5, 30 and 31 come from the stage file in tumpel/deploy. Rows 13, 20 and 29 go. The other rows from 6 to 28 derive from the stage file through the stack. Rows 32 to 34 live in tumpel/deploy or on the stage's hosts.

## Appendix B: where each top-level entry goes

| Entry                                                                                                         | Repository | Note                                                                                   |
| ------------------------------------------------------------------------------------------------------------- | ---------- | -------------------------------------------------------------------------------------- |
| `.claude/`                                                                                                    | Backend    | Infra and deploy keep their own short agent rules                                      |
| `.devcontainer/`                                                                                              | Backend    |                                                                                        |
| `.githooks/`                                                                                                  | Backend    | Infra and deploy carry their own pre-commit hook                                       |
| `.github/workflows/checks.yml`, `images.yml`, `release.yml`, `auto-merge.yml`                                 | Backend    | Moved to hosted runners in X2; `release.yml` publishes GitHub release assets           |
| `.github/workflows/tumpel-prebuild.yml`                                                                       | Deploy     | It names dev's API origin                                                              |
| `.gitignore`, `biome.json`, `package.json`, `package-lock.json`                                               | Backend    | Infra gets its own                                                                     |
| `AGENTS.md`, `CLAUDE.md`, `README.md`                                                                         | Backend    | Infra and deploy get their own                                                         |
| `LICENSE`, `NOTICE`                                                                                           | Backend    | Infra carries the same Apache-2.0 files                                                |
| `deploy/runner/`                                                                                              | Deploy     | Retires with the self-hosted runner (D18)                                              |
| `doc/adr/`, `doc/RFC/`, `doc/EDD/`, `doc/product/`                                                            | Backend    |                                                                                        |
| `doc/config.md`, `doc/surface.md`, `doc/guest-image.md`, `doc/github-actions-prebuild.md`, `doc/test-host.md` | Backend    |                                                                                        |
| `doc/deploy.md`                                                                                               | Infra      | Stage-specific parts (dev App, `tumpel.dev`, fixtures) move to deploy                  |
| `doc/personas/proof*`, `openrouter`, `ticket-worker`                                                          | Backend    | The proof and examples                                                                 |
| `doc/personas/tumpel-dev*`                                                                                    | Deploy     | The operator's own tenant config                                                       |
| `npm-scripts/deploy.mjs`                                                                                      | Infra      | Becomes the stage program's `deploy` and `teardown`                                    |
| `npm-scripts/proof-env.mjs`                                                                                   | Infra      | Reads the stage's tofu outputs                                                         |
| `npm-scripts/release-secrets.mjs`                                                                             | Deleted    | No Cloudflare secret in GitHub (D12)                                                   |
| Every other `npm-scripts/` file                                                                               | Backend    | Checks, release, guest image, harness layers, prebuild, proof, tenant and ticket tools |
| `packages/control-plane/` except `infra/`                                                                     | Backend    | `wrangler.jsonc` keeps local values only                                               |
| `packages/control-plane/infra/`                                                                               | Infra      | The stack                                                                              |
| `packages/dashboard-site/`                                                                                    | Backend    |                                                                                        |
| `packages/infra-worker/`                                                                                      | Deleted    | D16                                                                                    |
| `release-pins.json`                                                                                           | Backend    | A backend release fixes its supervisor and CLI                                         |
| `rules/`                                                                                                      | Backend    | Infra and deploy link to it                                                            |
| `cloudflare-builds-bootstrap`, `tofu-bootstrap` (external)                                                    | Deploy     | Builder projects and account bootstrap                                                 |
