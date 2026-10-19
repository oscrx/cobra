# Add `MarkFlagAppendEqual` to suggest `--flag=` in completions

## Problem

Flags that take an optional value (`NoOptDefVal`) only bind a value with the
`--flag=value` form; `--flag value` sets the flag to its default and treats
`value` as a positional argument. A common example is kubectl's `--dry-run`
(`--dry-run=client|server|none`).

Shell completion currently completes such a flag as `--flag ` (with a trailing
space), which steers users toward the form that does not work.

## Change

- New opt-in API `Command.MarkFlagAppendEqual(flagName string) error`, backed by
  the exported annotation key `FlagAppendEqualAnnotation`.
- `getFlagNameCompletions` suggests `--flag=` (instead of `--flag`) for flags
  carrying that annotation. Because it is opt-in per flag, it doesn't bring back
  the duplicate `--flag` / `--flag=` suggestions that caused the `=` form to be
  removed for all flags.
- Fixed the `ShellCompDirectiveNoSpace` detection for a single flag-name
  completion:
  - It now inspects only the completion choice, the text before the tab, not
    the description. Before, a completion with a description never matched
    `HasSuffix(..., "=")`.
  - It ORs `NoSpace` into the directive instead of replacing it, so
    `ShellCompDirectiveNoFileComp` is kept (resulting directive `:6`).

## Compatibility

This is fully backward compatible: behaviour changes only for flags explicitly
marked with `MarkFlagAppendEqual`. Before this change, the old `NoSpace` branch
could not be reached from cobra's own flag-name completions, so no existing
output changes.

## Usage

```go
cmd.Flags().String("dry-run", "none", `Must be "none", "server", or "client".`)
cmd.Flags().Lookup("dry-run").NoOptDefVal = "unchanged"
_ = cmd.MarkFlagAppendEqual("dry-run")
```

```
$ prog __complete --dry
--dry-run=	Must be "none", "server", or "client".
:6
```

## Tests

- Added `TestFlagNameCompletionAppendEqual`:
  - A marked flag completes as `--flag=` with directive `:6`
    (NoSpace + NoFileComp).
  - An unmarked flag still completes as `--flag` with directive `:4`.
  - `MarkFlagAppendEqual` on an unknown flag returns an error.
- `go test ./...` passes.
