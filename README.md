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

### Part 1: TypeScript LSP (tsgo) working on its own

The repo ships with `.claude/settings.json` pre-configured with only the tsgo
plugin enabled (graphql-lsp is disabled):

```json
{
  "enabledPlugins": {
    "graphql-lsp@graphql-analyzer": false,
    "tsgo@graphql-analyzer": true
  }
}
```

### 3. Run Claude Code

```sh
claude
```

### 4. Verify tsgo is NOT running yet

In a separate terminal, confirm no tsgo process has been spawned:

```sh
pgrep -fa tsgo
```

This should return nothing. LSP servers do **not** start automatically when
Claude Code launches — they only start when Claude invokes the LSP tool.

### 5. Ask Claude to invoke the LSP tool

Back in your Claude Code session, ask Claude to use the LSP tool:

```
use the LSP tool on src/index.ts
```

### 6. Verify tsgo IS running

In your separate terminal, check again:

```sh
pgrep -fa tsgo
```

This should now return a matching process. The tsgo LSP server is running and
serving TypeScript intelligence for `src/index.ts`.

### 7. Verify the LSP tool works on TypeScript files

Ask Claude to use the LSP tool on `src/index.ts`:

```
use the LSP hover tool on src/index.ts line 1, character 11 (imported `gql` tag)
```

Claude should successfully return hover information from tsgo (e.g. type info
for the `gql` import). This confirms the TypeScript LSP is working correctly.

## Part 2: Enabling the GraphQL plugin breaks tsgo

Now we'll enable the graphql-lsp plugin and observe that tsgo stops being used
for TypeScript files.

### 7. Enable the GraphQL plugin

Edit `.claude/settings.json` to enable the graphql-lsp plugin:

```json
{
  "enabledPlugins": {
    "graphql-lsp@graphql-analyzer": true,
    "tsgo@graphql-analyzer": true
  }
}
```

### 8. Restart Claude Code

Exit your current Claude Code session and start a new one:

```sh
claude
```

### 9. Verify neither LSP is running yet

In your separate terminal:

```sh
pgrep -fa tsgo
pgrep -fa graphql-lsp
```

Both should return nothing.

### 10. Ask Claude to invoke the LSP tool

```
use the LSP tool on src/index.ts
```

### 11. Check which LSP servers are running

```sh
pgrep -fa tsgo
pgrep -fa graphql-lsp
```

With both plugins enabled, the graphql-lsp process handles `.ts` files and
tsgo is **not** invoked for TypeScript files — even though both plugins are
enabled.

## Repo structure

```
.claude/settings.json   # LSP plugin configuration
.graphqlrc.yml           # GraphQL language server config (schema + documents)
schema.graphql           # GraphQL schema
src/index.ts             # Sample file using graphql-tag
tsconfig.json            # TypeScript config
```
