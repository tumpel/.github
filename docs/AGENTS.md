# Agent rules

Read [000. Tenets](https://github.com/tumpel/control-plane/blob/main/doc/adr/000-tenets.md) first. Its tenets bind every change.

Always follow [Errors](https://github.com/tumpel/control-plane/blob/main/rules/errors.md), [Coding standards](https://github.com/tumpel/control-plane/blob/main/rules/coding-standards.md) and [Architecture](https://github.com/tumpel/control-plane/blob/main/rules/architecture.md).

- When writing TypeScript, follow [TypeScript](https://github.com/tumpel/control-plane/blob/main/rules/typescript.md).
- When a change needs a new or changed type, follow [TypeSpec](https://github.com/tumpel/control-plane/blob/main/rules/typespec.md).
- When changing a table, a migration or a stored record, follow [Data modeling](https://github.com/tumpel/control-plane/blob/main/rules/data-modeling.md).
- When storing data, adding a retry or adding an endpoint, follow [Storage](https://github.com/tumpel/control-plane/blob/main/rules/storage.md).
- When adding or changing an operation, an MCP tool, a CLI command or the host channel, follow [API](https://github.com/tumpel/control-plane/blob/main/rules/api.md).
- When writing or changing a test, follow [Tests](https://github.com/tumpel/control-plane/blob/main/rules/tests.md).
- When committing or opening a pull request, follow [Commits and pull requests](https://github.com/tumpel/control-plane/blob/main/rules/commits-and-prs.md).
- When writing text a user reads, docs, an issue or a PR body, follow [Writing](https://github.com/tumpel/control-plane/blob/main/rules/writing.md).

## Commands

- `npm install` installs dependencies.
- `npm run check` runs every check. `npm run fix` formats.
- `npm test` runs the TypeScript tests.
