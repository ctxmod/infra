---
paths:
  - ".github/workflows/*"
  - ".github/workflows/*/*"
  - ".github/workflows/*/*/*"
---

# CI Workflow Conventions

A workflow runs when something it produces could have changed: derive its path
filter from the outputs-to-inputs map, not from convention, and include the
workflow's own path. CI verifies the committed state — auto-update and auto-fix
flags stay off, generated-output drift fails the run with a diff for a human to
produce locally and review. Script steps set strict modes and failures name the
failing field.

Every non-obvious step carries a comment stating *why* it exists; a step whose why
cannot be stated is a deletion candidate. Derived files are materialized by
explicit steps so a fresh clone builds with no undocumented local setup, and run
outputs land under run-scoped directories (run id, sha) so parallel runs never
overwrite each other.

- trigger derivation method, fail-loudly baseline, rationale-comment discipline, materialization ordering, and run-scoped artifact layout — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::delivery@core/reference/ci_workflows.md" />
