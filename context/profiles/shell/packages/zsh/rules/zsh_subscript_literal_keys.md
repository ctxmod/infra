# Generated zsh: subscripts and collisions

Rules for zsh emitted from templates (shell setup files, remote overlays,
shims) and for any generated shell that indexes `aliases`, `functions`, or
associative arrays by a runtime-derived name.

## Bare subscript names are literal keys in expansion forms

In zsh, inside a parameter *expansion* form (`${+arr[name]}`,
`${arr[name]}`, `${arr[name]+word}`), a bare name in the subscript is
treated as a **literal key**; only `$name` is expanded. This applies
to plain associative arrays and to the special parameters (`aliases`,
`functions`, …).

The asymmetry that makes this dangerous: **assignment subscripts**
(`aliases[name]=…`) DO expand bare names. A loop's writes therefore
work while its read-side guard silently never fires.

Worked failure: a cache-alias loop wrote `aliases[_name]` (expanded,
assignment side) but guarded with `(( ${+aliases[_name]} ))` (literal key,
expansion side), so the guard never fired. The loop overwrote every
generated alias and created an alias alongside a same-named function, and
any re-source of `~/.zshrc` then died with
`defining function based on alias '<name>'` at the function definition.

## Rules for generated shell

1. Any expansion-form subscript keyed by a runtime-derived name must
   carry `$` on the read/test side. Correct guard shape:

   ```zsh
   (( ${+aliases[$name]} || ${+functions[$name]} ))
   ```

2. Generated shell must skip function names when creating same-named
   aliases — zsh refuses `name() {` through an alias.

## Verification note

Agent tool output condensers may rewrite long diff/head displays with
literal `// … NNN lines omitted` markers. Verify deployed-file writes with
python/`cmp` assertions, not displayed diffs.

Generated output is never hand-edited to chase drift, and guards around
generated trees must match what the generator actually emits.
