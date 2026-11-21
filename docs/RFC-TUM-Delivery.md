# RFC: Delivery, build, release and deploy redesigned from first principles

| Field   | Value                                                                                                                                                                                                                                                       |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Status  | Proposed                                                                                                                                                                                                                                                    |
| Scope   | Design only; the release workflows of every component repository, the assembly and promotion jobs in tumpel/deploy, the supervisor updater, the Workers Builds projects, CI runners and Cloudflare tokens                                                   |
| Related | [RFC: Stages](RFC-TUM-Stages.md), [EDD-008: Supervisor self-update](008-supervisor-self-update.md), [EDD-009: Protocol compatibility](009-protocol-compatibility.md), [ADR 000: tenets](000-tenets.md) |

## Summary

Each component repository publishes immutable releases, built once, with a sha256 per asset. A job in tumpel/deploy assembles a release candidate, `rc-N`, from the newest release of each component and attaches its pins. A second job promotes one RC to one stage: it checks the bytes, applies infrastructure and migrations, uploads pre-built Worker versions, runs a smoke check and writes `current.json` last. A failed smoke check restores the previous Worker versions.

Every pointer has one writer. Versions only move forward, except through an explicit rollback, which is the previous RC promoted with `rollback=true`. Hosted runners run every job except the guest image build.

Fourteen PRs, ordered by risk, get there. The first four stop three of the six failures below.

## Problem statement

On 2026-11-20 six failures hit the pipeline.

1. tumpel/deploy pins supervisor `0.2.1-main.1`. Its release pushed that binary to `tumpel-host`, which downgraded with no error. The updater compares sha256 only and never compares versions.
2. mvhenten/tumpel and tumpel/deploy both write `current.json`. Three pin files name three different supervisors.
3. The supervisor release fails on a git dependency on `tumpel-api`, which `cargo publish` refuses. The supervisor has published no release since #74.
4. A control-plane pin bump deployed a tenant migration that exceeded the Durable Object CPU limit. Every signed-in call returned 500. Dev is the only stage, and tenant migrations run lazily inside the tenant Durable Object.
5. Workers Builds deploys on every push to tumpel/deploy main. A release and a deploy are two separate events.
6. The release workflow needed secrets that existed only on mvhenten/tumpel. `release-publish.mjs` orders releases by git ancestry, so it refused a `current.json` that named another repository's commit.

## What the code shows

| Part                        | Version scheme                                      | Builds                                                                                | Writes                                                                    |
| --------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| protocol                    | `vX.Y.Z-rc.N` on every main push, stable by promote | npm tarballs as release assets                                                        | `tumpel-api` to crates.io, stable only                                    |
| supervisor                  | RC on main push; older `-main.N` tags remain        | binaries, rebuilt on promote                                                          | GitHub release; crates on stable. Failing since #74                       |
| cli                         | RC on main push, promote                            | binary                                                                                | GitHub release                                                            |
| control-plane               | none                                                | nothing; tumpel/deploy builds it from a pinned commit                                 | its own `release-pins.json` (`0.2.1-rc.5`)                                |
| infra                       | none                                                | nothing; pinned by commit                                                             | nothing                                                                   |
| dashboard                   | `vX.Y.Z` tag                                        | tarballs                                                                              | GitHub release                                                            |
| deploy                      | `v0.2.N` tags, release branch                       | layers on every main push; the images job wants a `tumpel-build` runner the org lacks | `current.json` on every main push; pins supervisor `0.2.1-main.1`         |
| mvhenten/tumpel             | none                                                | nothing                                                                               | `current.json`; 5 self-hosted runners registered to it                    |
| cloudflare-builds-bootstrap | none                                                | nothing                                                                               | the `tumpel` and `tumpel-infra` Builds projects on deploy main, all paths |

## Principles

1. P1: One source of truth records what runs where.
2. P2: An artifact is versioned, immutable and built once.
3. P3: Promotion moves pointers and never rebuilds.
4. P4: Versions only move forward. Only an explicit rollback moves one back.
5. P5: Every pointer has one writer.
6. P6: Every stage runs a smoke check, and a failure blocks promotion.
7. P7: An urgent fix takes the same path as every other change.
8. P8: No code path depends on the stage.
9. P9: The database holds metadata only. Block storage lives in R2.

## Target design

### Component releases

Decision D1: each component repository publishes immutable GitHub releases, built once, with a sha256 per asset. Control-plane adds a bundle: the built Workers, their configs and the migrations. Infra adds a tarball. Guest images and layers go to content-addressed keys in R2.

### The RC assembly job

Decision D2: a job in tumpel/deploy picks the newest published release of each component, tags `rc-N` and attaches `pins.json`. It refuses a version older than the one in the previous RC, a `-main.N` build, an asset that lacks a sha256, and a channel or guest protocol range that does not match. tumpel/deploy main holds no versions. An urgent fix runs the job with `base=rc-N` and the one component to move.

### The promotion job

Decision D3: one job takes a stage and an RC and runs, in order:

1. Verify every sha256. Refuse an RC older than the stage's current one unless `rollback=true`.
2. Run `tofu apply`, copy the layers and apply D1 migrations.
3. Upload the pre-built Worker versions. The job rebuilds nothing.
4. Smoke check: signed-in calls against a seeded tenant, and `/v1/releases/current`. On failure, restore the previous Worker versions.
5. Write `current.json` last.

Every stage runs this job. Stages differ only in the stage file and the GitHub Environment.

### One writer per pointer

Decision D4: a pointer names the version a consumer runs. Each has one writer:

| Pointer                                     | Writer                             |
| ------------------------------------------- | ---------------------------------- |
| A component release                         | That repository's release workflow |
| `rc-N`                                      | The assembly job                   |
| Worker versions, tofu state, `current.json` | That stage's promotion job         |
| The binary on a host                        | That host's updater                |

The Workers Builds projects, the `release.yml` that writes on every push and `release-secrets.mjs` go away.

### Protocol crates

Decision D5: protocol publishes every `tumpel-api` RC to crates.io. Supervisor and cli pin it exactly, as `=0.21.0-rc.N`. Their PR check runs `cargo publish --dry-run`, so a git dependency fails before merge.

### Hosts

Decision D6: the supervisor refuses an older version unless the release manifest carries `rollback: true`.

### Rollback

Decision D7: a rollback promotes the previous RC with `rollback=true` and takes minutes. Workers switch versions at once, and hosts accept the older binary. Tenant migrations must be add-only, with no table rewrites, so the older Worker runs against the newer schema.

### Runners and secrets

Decision D8: hosted runners run every job. tumpel/deploy goes public, so its minutes are free. One org-level homelab runner builds the guest image, because that build needs KVM and root. It sits in a runner group limited to that workflow, which runs on tags or manual dispatch. A fork PR needs approval and never reaches it. Cloudflare tokens become per-stage GitHub Environment secrets that `tofu` bootstrap mints.

## Migration stages

Decision D9: each stage is one PR, ordered by risk. "Approval" marks a PR that touches CI or infrastructure and needs the owner's explicit approval.

| #   | Repository                  | Change                                                                                                        | Gate               | Approval  | Stops failure |
| --- | --------------------------- | ------------------------------------------------------------------------------------------------------------- | ------------------ | --------- | ------------- |
| 1   | tumpel/.github              | This RFC and the docs index                                                                                   |                    |           |               |
| 2   | supervisor                  | Refuse a strictly older version, with an updater unit test                                                    |                    |           | 1             |
| 3   | deploy                      | `release.yml` writes on tags or manual runs only; rotate the R2 keys so mvhenten/tumpel can no longer write   |                    | CI, infra | 2             |
| 4   | cloudflare-builds-bootstrap | The Builds projects build from a release tag                                                                  |                    | infra     | 5, for now    |
| 5   | protocol                    | Publish RC crates to crates.io                                                                                |                    | CI        |               |
| 6   | supervisor, cli             | Exact crates.io dependency and the publish dry run in checks                                                  | 5                  | CI        | 3             |
| 7   | control-plane               | A bounded tenant-migration test against a fixture tenant with history, and the add-only migration rule        |                    |           |               |
| 8   | control-plane, infra        | Release workflows that publish the bundle and the tarball                                                     |                    | CI        |               |
| 9   | deploy                      | The assembly job (in progress), with the refusals of D2                                                       | 6, 8               |           |               |
| 10  | org runners                 | Homelab runners at org level, scoped to the image workflow, fork PRs gated                                    |                    | infra, CI |               |
| 11  | deploy                      | The promotion job, smoke check, automatic Worker rollback and Environment secrets; delete the Builds projects | 9, RFC: Stages D16 | CI, infra | 4, 6          |
| 12  | deploy, infra               | A second stage with a seeded tenant                                                                           | 11                 | infra     |               |
| 13  | protocol, supervisor        | An additive `rollback` field in `ReleaseManifest` and the guarded downgrade                                   | 2                  |           |               |
| 14  | supervisor, cli, protocol   | Promote copies the RC's assets and stops rebuilding                                                           | open question 2    | CI        |               |

## Open questions

1. Second stage. Recommendation: dev stays the stage users sign in to. A new `staging` stage receives every RC automatically, and promotion to dev is manual.
2. Stable component releases. Recommendation: copy the RC's bytes under the stable tag and never rebuild.
3. Where pins live. Recommendation: an asset on the `rc-N` release in tumpel/deploy, with no pin file on main.
4. Supervisor crates on crates.io. Recommendation: publish only `tumpel-api`. The binary is what ships.
5. What starts the assembly job. Recommendation: a dispatch from each component release, plus manual runs. No schedule.
6. A failed smoke check. Recommendation: restore the previous Worker versions automatically.
7. Image layers. Recommendation: content-addressed objects in R2, with a manifest on the GitHub release. A release asset caps at 2 GiB, and the harness licences are unresolved.
8. RFC: Stages. Recommendation: this RFC replaces its D12 to D15 and its open question on deploy triggers. Its stack and stage config parts stay.

## Appendix of terms

| Term        | Meaning                                                       |
| ----------- | ------------------------------------------------------------- |
| RC          | `rc-N`, a tagged set of component versions in tumpel/deploy   |
| Pointer     | A record that names the version a consumer runs               |
| Promotion   | Moving one stage's pointers to one RC                         |
| Stage file  | The per-stage config in tumpel/deploy                         |
| Smoke check | Signed-in calls against a seeded tenant that gate a promotion |
