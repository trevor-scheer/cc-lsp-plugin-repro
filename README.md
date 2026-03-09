# cc-lsp-plugin-repro

## TL;DR

When multiple Claude Code LSP plugins declare interest in the same file
extension (e.g. both `tsgo` and `graphql-lsp` handle `.ts` files), only one
plugin is used — whichever is listed first in `enabledPlugins` wins. There
doesn't appear to be a way to have both plugins active for the same extension.

In practice, this means enabling the `graphql-lsp` plugin (listed first) causes
TypeScript files to be routed to graphql-lsp instead of tsgo, and TypeScript
intelligence is lost entirely. Reordering the plugins so tsgo is listed first
restores TypeScript intelligence, but then graphql-lsp never starts.

The expected behavior is that both plugins should be able to serve their
respective intelligence for `.ts` files simultaneously.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- Node.js and npm

## Setup

### 1. Clone and install dependencies

```sh
git clone https://github.com/trevor-scheer/cc-lsp-plugin-repro.git
cd cc-lsp-plugin-repro
npm install
```

### 2. Install the `graphql-lsp` binary

Download the appropriate binary for your platform from the
[graphql-analyzer-lsp releases](https://github.com/trevor-scheer/graphql-analyzer/releases?q=graphql-analyzer-lsp)
page (latest: v0.1.5).

For example, on macOS (Apple Silicon):

```sh
# Download and extract
curl -L https://github.com/trevor-scheer/graphql-analyzer/releases/download/graphql-analyzer-lsp%2Fv0.1.5/graphql-lsp-aarch64-apple-darwin.tar.xz | tar -xJ

# Move to a directory on your PATH
mv graphql-lsp /usr/local/bin/

# Verify
which graphql-lsp
```

Available platform binaries:
| Platform | Asset |
|---|---|
| macOS (Apple Silicon) | `graphql-lsp-aarch64-apple-darwin.tar.xz` |
| macOS (Intel) | `graphql-lsp-x86_64-apple-darwin.tar.xz` |
| Linux (ARM64) | `graphql-lsp-aarch64-unknown-linux-gnu.tar.xz` |
| Linux (x86_64) | `graphql-lsp-x86_64-unknown-linux-gnu.tar.xz` |
| Windows (x86_64) | `graphql-lsp-x86_64-pc-windows-msvc.zip` |

## Reproducing the issue

The easiest way to reproduce is to let Claude do it. After completing the setup
steps above, run `claude` and ask:

```
guide me through the LSP plugin reproduction
```

Claude knows the full reproduction plan (defined in `CLAUDE.md`) and will walk
you through all three parts, validating each step along the way.

If you prefer to run the steps manually, follow the instructions below.

---

## Manual reproduction

LSP servers do **not** start automatically when Claude Code launches — they only
start when Claude invokes the LSP tool. Each part verifies which servers start
by checking process counts before and after invoking the tool.

### Part 1: tsgo works on its own

The repo ships with `.claude/settings.json` pre-configured with only tsgo
enabled:

```json
{
  "enabledPlugins": {
    "graphql-lsp@graphql-analyzer": false,
    "tsgo@graphql-analyzer": true
  }
}
```

1. Run `claude` in the repo.
2. In a separate terminal, verify no Claude-owned tsgo is running yet:
   ```sh
   pgrep -fa tsgo
   ```
3. Ask Claude to invoke the LSP tool:
   ```
   use the LSP hover tool on src/index.ts line 1, character 11
   ```
4. Check processes again — tsgo should now be running. The hover should return
   TypeScript type info for the `gql` import (e.g. `function gql(...): DocumentNode`).

### Part 2: enabling graphql-lsp breaks tsgo

1. Update `.claude/settings.json` — enable graphql-lsp, listed **first**:
   ```json
   {
     "enabledPlugins": {
       "graphql-lsp@graphql-analyzer": true,
       "tsgo@graphql-analyzer": true
     }
   }
   ```
2. Restart Claude Code (`/exit`, then `claude`).
3. In a separate terminal, verify neither LSP is running yet:
   ```sh
   pgrep -fa tsgo
   pgrep -fa graphql-lsp
   ```
4. Ask Claude to invoke the LSP tool on the same position:
   ```
   use the LSP hover tool on src/index.ts line 1, character 11
   ```
5. Check processes again — graphql-lsp should be running, but tsgo should
   **not** be. The hover result will differ from Part 1 (no TypeScript type
   info). TypeScript intelligence is lost.

### Part 3: plugin ordering determines the outcome

1. Update `.claude/settings.json` — keep both enabled, but list **tsgo first**:
   ```json
   {
     "enabledPlugins": {
       "tsgo@graphql-analyzer": true,
       "graphql-lsp@graphql-analyzer": true
     }
   }
   ```
2. Restart Claude Code.
3. Verify neither LSP is running yet.
4. Ask Claude to invoke the LSP tool on the same position:
   ```
   use the LSP hover tool on src/index.ts line 1, character 11
   ```
5. Check processes — this time tsgo should be running and graphql-lsp should
   **not** be. The hover returns TypeScript type info again, same as Part 1.

This suggests the order of plugins in `enabledPlugins` determines which server
handles a given file extension — the first listed plugin appears to win, and
only one plugin is used per extension.

## Repo structure

```
.claude/settings.json   # LSP plugin configuration
.graphqlrc.yml           # GraphQL language server config (schema + documents)
schema.graphql           # GraphQL schema
src/index.ts             # Sample file using graphql-tag
tsconfig.json            # TypeScript config
```
