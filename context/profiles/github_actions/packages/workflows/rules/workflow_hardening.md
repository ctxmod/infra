---
paths:
  - ".github/workflows/*"
  - ".github/workflows/*/*"
  - ".github/workflows/*/*/*"
  - ".github/actions/*/*"
  - ".github/actions/*/*/*"
---

# Workflow Hardening

Lock every action a workflow runs — a `.github/workflows/actions.lock` from
`gh actions-lock` where the lockfile policy is available, otherwise a full
commit SHA with the version as a trailing comment — and pin reusable-workflow
calls by SHA, because the lockfile does not cover them. Default
`permissions:` to `{}` or `contents: read` and raise scopes per job;
`id-token: write` only on jobs that federate. Never expand untrusted event
fields with `${{ }}` inside `run:` — pass them through `env:`.

Treat `pull_request_target` and PR-triggered `workflow_run` as privileged:
never check out or execute the fork's code there, and expect public
repositories to block `pull_request_target` from 2026-12-02 unless a policy
allows it. Gate every workflow change on both actionlint and zizmor.

- lockfiles and SHA-pinning policy, token permissions, script injection, the 2025–2026 `pull_request_target` changes, OIDC numeric claims, attestations and immutable releases, and the two linters — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::workflows@github_actions/reference/workflow_hardening.md" />
