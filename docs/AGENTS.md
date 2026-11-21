# Agent rules

Read [000. Tenets](000-tenets.md) first. Its tenets bind every change.

Always follow [Errors](errors.md), [Coding standards](coding-standards.md) and [Architecture](architecture.md).

- When writing TypeScript, follow [TypeScript](typescript.md).
- When a change needs a new or changed type, follow [TypeSpec](typespec.md).
- When changing a table, a migration or a stored record, follow [Data modeling](data-modeling.md).
- When storing data, adding a retry or adding an endpoint, follow [Storage](storage.md).
- When adding or changing an operation, an MCP tool, a CLI command or the host channel, follow [API](api.md).
- When writing or changing a test, follow [Tests](tests.md).
- When committing or opening a pull request, follow [Commits and pull requests](commits-and-prs.md).
- When writing text a user reads, docs, an issue or a PR body, follow [Writing](writing.md).

## Commands

- `npm install` installs dependencies.
- `npm run check` runs every check. `npm run fix` formats.
- `npm test` runs the TypeScript tests.
