# cc-lsp-plugin-repro

Reproduction repo demonstrating that enabling the graphql-lsp plugin prevents
tsgo from being used for TypeScript files, even when both plugins are enabled.

## Repo structure

```
.claude/settings.json   # LSP plugin configuration (controls which plugins are enabled)
.graphqlrc.yml           # GraphQL LSP config — schema: schema.graphql, documents: src/**/*.ts
schema.graphql           # Simple GraphQL schema (Query.hello)
src/index.ts             # Sample TS file importing gql from graphql-tag
tsconfig.json            # TypeScript config
```

## LSP plugin configuration

`.claude/settings.json` controls which plugins are active:
- `tsgo@graphql-analyzer` — TypeScript language server (tsgo)
- `graphql-lsp@graphql-analyzer` — GraphQL language server

LSP servers do NOT start when Claude Code launches. They only start when Claude
invokes the LSP tool on a matching file.

**Note:** VS Code (or other editors) may have their own tsgo/graphql-lsp
processes running independently. These are unrelated to Claude Code's LSP
servers. When validating, count the number of processes **before** and **after**
invoking the LSP tool — a new process appearing confirms Claude Code started its
own server. Don't try to kill editor-owned processes; they will respawn.

## Reproduction guide

This reproduction has two parts. Follow each step in order. Use the checklist
below to track progress across sessions.

### Part 1: tsgo works on its own

**Precondition:** `.claude/settings.json` must have graphql-lsp DISABLED:
```json
{
  "enabledPlugins": {
    "graphql-lsp@graphql-analyzer": false,
    "tsgo@graphql-analyzer": true
  }
}
```

If it doesn't match, update it before proceeding.

1. **Snapshot LSP processes** — Run `pgrep -fc tsgo` and `pgrep -fc graphql-lsp`
   to get a count of existing processes (may be >0 if an editor is running).
   Record these counts as the baseline.
2. **Invoke the LSP tool** — Use `LSP hover` on `src/index.ts` line 1, character
   11 (the `gql` import). This triggers tsgo to start. If the server is still
   starting, wait a few seconds and retry.
3. **Validate tsgo count increased** — Run `pgrep -fc tsgo` again. The count
   should be baseline + 1. `pgrep -fc graphql-lsp` should be unchanged.
4. **Verify LSP results** — The hover should return type info for the `gql` import
   from graphql-tag. This confirms tsgo works for TypeScript files.
5. **Prompt user** — Tell the user: "Part 1 complete. tsgo is working for TypeScript
   files. Now we need to enable the graphql-lsp plugin and restart Claude Code to
   reproduce the issue. I'll update the settings — please exit Claude Code (`/exit`)
   and restart it (`claude`) after I do."
6. **Enable graphql-lsp** — Update `.claude/settings.json` to set
   `graphql-lsp@graphql-analyzer` to `true`.

### Part 2: enabling graphql-lsp breaks tsgo for TypeScript

**Precondition:** `.claude/settings.json` must have BOTH plugins enabled:
```json
{
  "enabledPlugins": {
    "graphql-lsp@graphql-analyzer": true,
    "tsgo@graphql-analyzer": true
  }
}
```

7. **Snapshot LSP processes** — Run `pgrep -fc tsgo` and `pgrep -fc graphql-lsp`
   to get baseline counts (fresh session after restart).
8. **Invoke the LSP tool** — Use `LSP hover` on `src/index.ts` line 1, character
   11 (same as before). If the server is still starting, wait a few seconds and
   retry.
9. **Validate which LSP counts changed** — Run `pgrep -fc tsgo` and
   `pgrep -fc graphql-lsp` again.
   - **Expected bug:** graphql-lsp count increased, tsgo count did NOT increase.
     The graphql-lsp has claimed `.ts` files and tsgo is never invoked, even
     though both plugins are enabled.
10. **Verify LSP results** — The hover result will likely differ from Part 1 (may
    return GraphQL info instead of TypeScript type info, or nothing useful).
11. **Summarize** — Explain the bug: with both plugins enabled, graphql-lsp handles
    `.ts` files and tsgo is never started. TypeScript intelligence is lost.

## Reproduction checklist

Track progress across sessions. Update this as each step completes.

- [ ] Settings configured for Part 1 (graphql-lsp: false, tsgo: true)
- [ ] Part 1: Recorded baseline LSP process counts
- [ ] Part 1: Invoked LSP tool on src/index.ts
- [ ] Part 1: Validated tsgo count increased after invocation
- [ ] Part 1: Confirmed hover returns TypeScript type info
- [ ] Settings updated for Part 2 (graphql-lsp: true, tsgo: true)
- [ ] Claude Code restarted for Part 2
- [ ] Part 2: Recorded baseline LSP process counts
- [ ] Part 2: Invoked LSP tool on src/index.ts
- [ ] Part 2: Validated which LSP counts changed after invocation
- [ ] Part 2: Confirmed tsgo count did NOT increase (bug reproduced)
- [ ] Part 2: Documented LSP hover result difference
