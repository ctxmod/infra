# GitHub Actions facts

GitHub Actions facts from CI work — setup-uv tag resolution, the numeric OIDC
environment subject, skip-vs-fail needs semantics, the fail-open summary fix
pattern, reusable-call result poisoning, static concurrency groups, structural
workflow tests, and Claude plugin CI without auth.

Verified GitHub Actions behavior, each costing a red run or a wrong diagnosis
before it was settled. Check here before re-deriving any of it in a workflow.
The general CI workflow conventions are
<ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::delivery@core/reference/ci_workflows.md" />.

## `uses:` refs — resolve them, actionlint does not

`astral-sh/setup-uv` publishes floating major tags only up to `v7`; from
v8 on only full versions exist (`v10.0.0` … `v10.2.0` as of 2026-10-01;
re-checked 2026-11-22: still no `v8`/`v9`/`v10` major tags, latest
`v10.4.0`).
`uses: astral-sh/setup-uv@v10` therefore fails at "Set up job" — and
**actionlint exits 0 on it**, because it does not resolve tags. Found
while validating a repository's CI (2026-10-01): a phase report claimed
actionlint-clean CI that could never have gone green. When validating any
workflow, resolve every `uses:` ref against the GitHub API
(`/repos/<o>/<r>/git/ref/tags/<tag>` and `/branches/<tag>`), not just
actionlint. Pin setup-uv by full version (`@v10.4.0`), or better by SHA or
through a workflow lockfile — the hardening reference
<ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::workflows@github_actions/reference/workflow_hardening.md" />
owns the pinning policy.

## OIDC — the numeric environment subject

GitHub Actions OIDC tokens for jobs in a GitHub environment now present
the subject in a numeric form alongside the classic one (verified live
2026-10-03, on a job with `environment: <name>`):

- Classic (documented): `repo:<owner>/<repo>:environment:<name>`
- Also presented: `repo:<owner>@<owner-user-id>/<repo>@<repo-id>:environment:<name>`
  — the owner's immutable user-ID and the repository ID (confirmed via
  the GitHub API).

The mismatch surfaces as Entra **AADSTS700213** ("No matching federated
identity record") on `azure/login` and any OIDC-exchanging action while
the workflow itself is correct. **Trust BOTH forms** — immutable IDs are
stable, so both are deterministic. Worked fix: a second
`azuread_application_federated_identity_credential` on the CD app (in the
bootstrap layer). Verify the presented subject from the AADSTS700213 error
text itself — it quotes the exact subject to add. Consequence: deleting and
recreating the repository changes the repo ID, so the federated credential
must be re-created.

## Skip vs fail semantics and the fail-open summary

- **A *skipped* needed job does NOT block a dependent whose own `if:` is
  true** — a real bypass risk. A *failed* one does block, unless the
  dependent's `if:` carries `always()`/`failure()`.
- **A summary job with `always() && !failure() && !cancelled()` over
  selection-gated needs runs on all-skipped pushes and publishes a null
  manifest — fail-open, green run.** On workflow-only pushes (paths match
  the trigger but no component case arm), every selection-gated
  reusable-call job skips, and `needs.<job>.outputs.*` of a skipped job
  resolve to EMPTY STRINGS, not errors; `always()` treats skipped as
  runnable and empty strings pass writer-less checks silently. This
  shipped a null release-identity artifact twice (two attempts of one run,
  byte-identical) before diagnosis.
- **Verified fix pattern** (in a `release.yml`): gate the
  summary on `needs.changes.result == 'success'` AND an OR-disjunction
  over the selection outputs, RETAINING the status terms — dropping them
  propagates the summary skip to production legs on mixed-selection
  pushes. Assert per-selected-component evidence (non-empty
  digest/version, executed-staging literals) with `exit 1` BEFORE
  upload-artifact. Self-describe the manifest (`selected_components` +
  `component_status` ∈ released | skipped_not_selected |
  skipped_not_configured) so optional skips are distinguishable from
  failures; "not configured" must be a plumbed `workflow_call` output
  (e.g. `staging_status`), not inferred from emptiness. Pin it with a
  hermetic structural test. Live proof shape for a workflow-only push:
  green run, all release legs + summary + production legs skipped,
  artifacts total_count 0, zero deployments for the head sha,
  `pending_deployments []`.

## Reusable-call needs-result poisoning

A job gating on `needs.<x>.result == 'success'` where `<x>` is a
reusable-workflow call (`uses: ./.github/workflows/...`) can NEVER pass
when the called workflow contains any job whose `if:` evaluates false
under the caller's event — GitHub evaluates the caller-level result
against the called workflow's individual jobs, and an internally skipped
job makes the aggregate non-success
(github/orgs/community/discussions/189172). Verified empirically
2026-10-06: the full release chain succeeded, but deploy-production skipped
at the gate because the called `app.yml`'s `build` job
(`if: github.event_name == 'pull_request'`) always skips under
workflow_dispatch — deploy-production had never executed since its
introduction. The same latent poisoning sat in two other called workflows
(an `infra-candidate.yml` with a PR-only `static-checks` job and an
`ingest-candidate.yml` with schedule-only `scheduled-eval`/`push` jobs).

**How to apply:** never let a production/deploy leg gate on
`needs.<reusable-call>.result`. Mirror the fail-closed summary shape
instead: `if: always() && !failure() && !cancelled() && <trust guards> &&
needs.changes.outputs.<component> == 'true' &&
needs.release-summary.result == 'success'` — the evidence-summary job
carries the real guarantee, and the component-selection predicate replaces
the selection semantics the skipped caller job used to provide. Keep the
reusable call in `needs` when outputs are consumed (outputs export fine
regardless of the poisoned aggregate result). Run/skip matrix tests that
model the reusable-call result as a clean boolean cannot see this class —
structural assertions (no production leg references a reusable-call
result) are needed.

## Static concurrency group for ingest-style workflows

An ingest workflow that re-derives a knowledge base carries a
workflow-level `concurrency: { group: ingest, cancel-in-progress: false }` —
a static group name so act evaluates it identically, deliberately deviating
from the repository's job-level `app-*- ${{ github.ref }}` groups.
**Serialization without cancellation** because a mid-`push_kb` cancel can
leave the index half converged and every run fully re-derives the knowledge
base: the stale-document sweep (read existing ids → delete) is non-atomic,
and overlapping runs can delete each other's fresh uploads while sharing the
capacity-1 embedding deployment.

## Workflow structural tests

Structural workflow tests live as shell scripts using the PyYAML path
with a grep fallback, enumerating event gates across the event set
(e.g. a `scripts/tests/test-ingest-workflow.sh` covering 4 events × act).
Run **negative controls before trusting structural tests** — a gate
mutation that is a silent no-op proves nothing. Where direct `actionlint`
is unavailable and a substitute lint runs instead, disclose the
substitute; never claim actionlint ran.

## Claude plugin CI without credentials

Verified 2026-11-03:

- `npm install -g @anthropic-ai/claude-code@2.1.317` works on
  ubuntu-latest. With an empty HOME and no API key, both
  `claude plugin validate plugin` and `claude plugin test plugin` exit 0;
  the test kit (`claude-code/testing`) ships inside the binary.
- The plugin API declarations live at
  `plugin/.claude-plugin/types/claude-code/index.d.ts`. They are
  gitignored and written by the engine "when it loads the mod from a
  folder the person owns" — and **none** of the following wrote them: an
  unauthenticated `-p` session, a `-p` session with a fake key, an
  authenticated `-p` session in a /tmp clone, `plugin init`. So
  `tsc -p plugin` fails in a fresh checkout (TS7006, "Cannot find module
  'claude-code/testing'").
- **How to apply:** gate plugins with `validate` and `test` in CI; run
  `tsc` on the dev host, where the engine has laid the types. Do not
  commit the engine types — they are written per build. The tsc step
  elides **loudly** when the types file is absent: a `::warning`
  annotation plus a `$GITHUB_STEP_SUMMARY` entry, with the step still
  running tsc whenever the file exists — never a silent skip, never a
  step that reds forever.

## Sources

- https://github.com/astral-sh/setup-uv/tags — tag list showing majors `v1`–`v7` only and `v10.4.0` latest (accessed 2026-11-22 via `git ls-remote --tags`)
- https://docs.cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines — numeric `*_id` claims recommended over names (accessed 2026-11-22)
