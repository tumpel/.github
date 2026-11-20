# Tumpel

Tumpel is a distributed agent operating system that lets you direct agent swarms on your own hardware, in full microVM isolation.

## Try it

The demo is at https://dev.tumpel.dev. Tumpel is in full open-source mode right now, so hosted Tumpel isn't taking open sign-ups. Anyone who wants to try it can ask for a preview by [getting in touch](https://github.com/tumpel/.github/issues/new).

## Tenets

1. No logic in the supervisor. The supervisor and the guest pass input through, and the control plane owns harness-specific behaviour.
2. Tumpel relies on what the harness itself does. The harness runs the agent loop, and Tumpel runs the harness.
3. No implicit automation. Nothing runs or spends tokens unless you configured it.
4. TypeSpec is the single source of truth. Generated output is never written by hand.
5. Every input surface has a versioned schema: each API operation, MCP tool, CLI argument and config file.
6. What can be created can be read back, through the API, MCP and the CLI. Only secret values stay write-only.
7. CI time is priority zero. Every change keeps cutting it.
8. Cost is a signal. A rising bill points at a design problem, and the design changes until the bill falls.
9. Parity across services. The dashboard, MCP, API and CLI share the same feature set.

## What it does

- MCP first. Tumpel was built and bootstrapped from a simple MCP all the way up to a fully functioning service, giving you a complete and well-defined MCP that is capable of rebuilding Tumpel on its own.
- Any harness with any subscription. Personas combine any subscription or token source with any harness, so you pick and match the right agent and subscription for the job.
- Self-hosted. Run on your own hardware, whether an OVH box, a recycled box, a Hetzner box, an EC2 instance or the desktop under your desk.
- Control plane and data plane separated by design. The full control plane runs on Cloudflare Workers at near-zero cost per month.
- Open source and self-deployable.
- Built from first principles.
- Real microVM isolation.
- Caching (prebuilt images) and a dashboard built in. See cost, and orchestrate and direct work from inside Tumpel.

## Repositories

- [control-plane](https://github.com/tumpel/control-plane): the Cloudflare Worker API that runs agent sessions.
- [supervisor](https://github.com/tumpel/supervisor): the host supervisor, its guest agent and the host channel crates.
- [protocol](https://github.com/tumpel/protocol): TypeSpec for the API and host channel, with the generated Rust crate and npm packages.
- [cli](https://github.com/tumpel/cli): the command line client.
- [dashboard](https://github.com/tumpel/dashboard): the web dashboard and the shared page shell.
- [infra](https://github.com/tumpel/infra): OpenTofu infrastructure on Cloudflare.
- [workflows](https://github.com/tumpel/workflows): reusable GitHub Actions workflows for prebuilds.
- [prebuild-fixture](https://github.com/tumpel/prebuild-fixture): a fixture repository for the end-to-end proof.
