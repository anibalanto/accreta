---
name: stratum-paths
description: "Whenever you need to reference a filesystem path in a shell command, resolve it through `$(stratum '<path>')`. Never hardcode absolute paths. Load this skill when working with stratum path tokens (`*`, `<`, `>name`, etc.) or when composing Bash commands that navigate stratum layers."
---

Whenever you need to reference a filesystem path in a shell command, resolve it through `$(stratum '<path>')`. Never hardcode absolute paths.

## Token syntax

| Token | Resolves to |
|-------|-------------|
| `*` | Outermost `.git` ancestor (project root) |
| `<*` | Nearest `.git` ancestor |
| `<` | Up one stratum layer (`../..`) |
| `<<` | Up two stratum layers, etc. |
| `>name` | Down into `.stratum/name` |
| `/segment` | Regular path component (joined after preceding token) |

Tokens compose left to right. Examples:

| Expression | Meaning |
|------------|---------|
| `*` | project root |
| `*/subsystems/stratum` | `stratum` subsystem spec layer |
| `*/subsystems/stratum>impl` | stratum impl layer |
| `*/subsystems/stratum>impl/crates/stratum/src` | path inside impl layer |
| `<` | parent spec layer from current |
| `>impl/crates` | impl layer, then into `crates/` |

## Rules

- Every `Bash` tool call that references a directory or file path must use `$(stratum '...')`.
- Use `stratum pws` to find the stratum path of the current working directory.
- Prefer `*`-rooted paths (from outermost git root) for clarity; use `<`/`>` for relative navigation within a layer.
- When composing a `git -C` command, always write: `git -C $(stratum '*/subsystems/foo>impl') ...`

## Quick reference

```bash
# Current stratum location
stratum pws

# Resolve a path
echo $(stratum '*/subsystems/stratum>impl')

# Use in commands
cat $(stratum '*/subsystems/stratum>impl/crates/stratum/src/lib.rs')
git -C $(stratum '*/subsystems/stratum>impl') log --oneline -5
cargo test --manifest-path $(stratum '*/subsystems/stratum>impl/Cargo.toml')
```
