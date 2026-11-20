<p align="center"><img src="assets/mascot.svg" alt="The tumpel salamander" width="160"></p>

# tumpel

Harnesses on any hardware, driven over MCP.

Tumpel has three layers:

- **Control plane**: a Cloudflare Worker that holds tenants, hosts and sessions, with one SQLite database per tenant.
- **Hosts**: machines a tenant owns, each running `tumpel-supervisor`, which boots a box per session and runs a harness (Claude Code, Codex, OpenCode or Pi.dev; Mistral's coding agent is planned) turn by turn.
- **Clients**: the `tumpel` CLI, a client of the same public API as the MCP server hosted on the control plane ([005. The MCP server is hosted](005-hosted-mcp.md)), so a terminal or an agent can start sessions and read their turns.

The API is defined once in TypeSpec, whose releases publish the `@tumpel/schemas`, `@tumpel/openapi` and `@tumpel/api` packages.

