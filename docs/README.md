# Tumpel docs

Design and product docs for Tumpel.

## Product

- [Dashboard requirements](dashboard.md)
- [GitHub connections](github-multi-install.md)
- [Repository picker](repository-picker.md)
- [Triggers: product review and configuration UX](triggers.md)

## Guides

- [Tenant config](config.md)
- [GitHub Actions prebuilds](github-actions-prebuild.md)
- [Guest image](guest-image.md)
- [Public surface](https://github.com/tumpel/control-plane/blob/main/doc/surface.md)

## RFCs

- [RFC: Connectors](RFC-TUM-Connectors.md)
- [RFC: Delivery, build, release and deploy redesigned from first principles](RFC-TUM-Delivery.md)
- [RFC: Events, one ordered log of domain events per tenant](RFC-TUM-Events.md)
- [RFC: Foreman, a session that turns its crew's events into its own turns](RFC-TUM-Foreman.md)
- [RFC: Prebuilds and build providers](RFC-TUM-Prebuilds.md)
- [RFC: Stages, one infrastructure definition instantiated per stage](RFC-TUM-Stages.md)
- [RFC: Tenant API, a versioned contract for the tenant Durable Object](RFC-TUM-TenantApi.md)
- [RFC: Triggers, sessions that start from an external event](RFC-TUM-Triggers.md)
- [RFC: Tumpel, harnesses on any hardware](RFC-TUM-Tumpel.md)

## EDDs

- [EDD-001: Runtime](001-runtime.md)
- [EDD-002: Host channel](002-host-channel.md)
- [EDD-003: Credentials](003-credentials.md)
- [EDD-004: Snapshots and transcripts](004-snapshots-and-transcripts.md)
- [EDD-005: MCP and CLI](005-mcp-and-cli.md)
- [EDD-006: The proof](006-end-to-end-proof.md)
- [EDD-007: Guest layers](007-guest-layers.md)
- [EDD-008: Supervisor self-update](008-supervisor-self-update.md)
- [EDD-009: Protocol compatibility](009-protocol-compatibility.md)

## ADRs

- [000. Tenets](https://github.com/tumpel/control-plane/blob/main/doc/adr/000-tenets.md)
- [001. Control plane on Cloudflare](https://github.com/tumpel/control-plane/blob/main/doc/adr/001-control-plane-on-cloudflare.md)
- [002. Provider keys follow personas, secret sets and dispatch credentials](https://github.com/tumpel/control-plane/blob/main/doc/adr/002-multi-harness-provider-keys.md)
- [003. The host channel dials out and is defined before any backend](https://github.com/tumpel/control-plane/blob/main/doc/adr/003-host-channel-dials-out-provider-agnostic.md)
- [004. TypeSpec to Rust via secondary tooling](https://github.com/tumpel/control-plane/blob/main/doc/adr/004-typespec-to-rust-via-secondary-tooling.md)
- [005. The MCP server is hosted](https://github.com/tumpel/control-plane/blob/main/doc/adr/005-hosted-mcp.md)
- [006. CircleCI connects through OAuth with PKCE and a loopback redirect](https://github.com/tumpel/control-plane/blob/main/doc/adr/006-circleci-oauth-loopback.md)
- [007. Connector tokens reach harnesses through a guest-local forwarder](https://github.com/tumpel/control-plane/blob/main/doc/adr/007-connector-forwarder.md)
- [007. A turn's goal runs on the harness's goal feature](https://github.com/tumpel/control-plane/blob/main/doc/adr/007-turn-goal-on-harness-goal-feature.md)
- [008. Connectors register OAuth clients through dynamic client registration only](https://github.com/tumpel/control-plane/blob/main/doc/adr/008-connector-dynamic-client-registration.md)
- [008. A snapshot follows every turn](https://github.com/tumpel/control-plane/blob/main/doc/adr/008-snapshot-after-every-turn.md)
- [009. Every turn's GitHub token reaches every installation](https://github.com/tumpel/control-plane/blob/main/doc/adr/009-turn-token-reaches-every-installation.md)
- [010. The database holds metadata, and bulk data lives in R2](https://github.com/tumpel/control-plane/blob/main/doc/adr/010-bulk-data-out-of-the-database.md)

## Personas

- [The OpenRouter personas](personas-openrouter-README.md)
- [Proof persona](personas-proof-tenant-CLAUDE.md)
- [Working a ticket](personas-ticket-worker-CLAUDE.md)
- [The ticket-worker persona](personas-ticket-worker-README.md)

## Rules

- [API](https://github.com/tumpel/control-plane/blob/main/rules/api.md)
- [Architecture](https://github.com/tumpel/control-plane/blob/main/rules/architecture.md)
- [Coding standards](https://github.com/tumpel/control-plane/blob/main/rules/coding-standards.md)
- [Commits and pull requests](https://github.com/tumpel/control-plane/blob/main/rules/commits-and-prs.md)
- [Data modeling](https://github.com/tumpel/control-plane/blob/main/rules/data-modeling.md)
- [Errors](https://github.com/tumpel/control-plane/blob/main/rules/errors.md)
- [Storage](https://github.com/tumpel/control-plane/blob/main/rules/storage.md)
- [Tests](https://github.com/tumpel/control-plane/blob/main/rules/tests.md)
- [TypeScript](https://github.com/tumpel/control-plane/blob/main/rules/typescript.md)
- [TypeSpec](https://github.com/tumpel/control-plane/blob/main/rules/typespec.md)
- [Writing](https://github.com/tumpel/control-plane/blob/main/rules/writing.md)

## Agent instructions

- [architect](architect.md)
- [explorer](explorer.md)
- [reviewer](reviewer.md)
- [writer](writer.md)
- [Agent rules](AGENTS.md)
- [CLAUDE](CLAUDE.md)

## Overview

- [tumpel](overview.md)
