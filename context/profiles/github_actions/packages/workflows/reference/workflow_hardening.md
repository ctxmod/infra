# GitHub Actions workflow hardening

GitHub Actions workflow hardening — workflow lockfiles and full-SHA pinning,
least-privilege `GITHUB_TOKEN` permissions, script injection, the
`pull_request_target` changes of 2025–2026, OIDC subject claims, artifact
attestations and immutable releases, and static analysis with zizmor and
actionlint. Dated platform facts as of 2026-11-22; re-verify before relying on
a date.

A workflow is code that runs with the repository's credentials, and every
`uses:` line imports someone else's code into that context. The controls below
are ordered by how much risk each removes per unit of effort.

## Lock what runs: lockfiles, then SHA pins

- **Workflow lockfiles (GA 2026-10-28).** The `gh actions-lock` GitHub CLI
  extension writes `.github/workflows/actions.lock`, recording the exact
  commit of every action an onboarded workflow uses, **including transitive
  dependencies** of composite actions, plus the source repository's owner and
  repository IDs. At run time GitHub executes the locked commit, requires it to
  be reachable from the recorded branch or tag (a fork-only commit is
  rejected), and checks the IDs, so a moved tag or a re-created repository
  under the same name cannot substitute code. GitHub's secure-use guide calls
  the lockfile the strongest available protection and says it supersedes SHA
  pinning for repositories that adopt it. Workflows keep readable tag
  references; Dependabot version updates regenerate the matching lock entry in
  the same pull request.
- **What the lockfile does not cover.** Reusable-workflow calls
  (`jobs.<id>.uses`) are not pinned by the caller's lockfile — reference them
  by full commit SHA; their own dependencies are locked in the repository that
  defines them. Inputs, secrets, and the runner image are outside its scope.
- **Enforce it.** Enterprise, organization, and repository admins can turn on
  the "Require lockfile" policy. Add a CI step that verifies the lockfile is
  current (the extension's verify mode) so drift fails a pull request rather
  than a release. The extension itself is still 0.x — pin its version.
- **Without a lockfile, pin by full commit SHA.** A tag can be moved or
  deleted by anyone who gains write access to the action's repository; a full
  40-character SHA from the action's own repository (not a fork) cannot. Keep
  the human-readable version as a trailing comment
  (`uses: actions/checkout@<sha> # v7.0.1`) so Dependabot can still propose
  updates. Since 2025-08 the allowed-actions policy can **require** full-SHA
  pinning (unpinned uses then fail the run) and can block a specific action or
  version with a `!` prefix entry. Dependabot *alerts* do not cover SHA-pinned
  actions, so pair pins with Dependabot *version updates*.
- **Resolve refs; do not trust a linter's silence.** actionlint does not
  resolve tags (see the `uses:` ref fact in
  <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::workflows@github_actions/reference/actions_facts.md" />).

## Least-privilege `GITHUB_TOKEN`

Set the organization or repository default token permission to read-only, then
declare `permissions:` explicitly — at workflow level as `{}` or
`contents: read`, raised per job only where the job needs it:

```yaml
permissions: {}            # nothing by default

jobs:
  release:
    permissions:
      contents: read
      id-token: write      # OIDC to the cloud and keyless signing
      attestations: write  # artifact attestations
      packages: write      # push to GHCR
```

`id-token: write` lets the job mint OIDC tokens; grant it only to jobs that
federate. Set `persist-credentials: false` on `actions/checkout` unless a later
step pushes with the checkout credential, so the token is not left in
`.git/config` for every subsequent step and uploaded artifact.

## Script injection

Never interpolate attacker-controllable context (`github.event.issue.title`,
`github.event.pull_request.head.ref`, `github.head_ref`, commit messages,
comment bodies) directly into `run:` — `${{ }}` is expanded into the script
text before the shell parses it. Pass the value through an environment
variable, which the shell treats as data:

```yaml
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: printf '%s\n' "$PR_TITLE"
```

The same applies to `actions/github-script` `script:` bodies and to any input
an action evals.

## `pull_request_target` and `workflow_run`

These triggers run with the base repository's secrets and a write-capable
token while the pull request comes from a fork. GitHub's rule: workflows on
these triggers must not check out or execute untrusted pull request code. Prefer
`pull_request` (no secrets for forks) plus a separate `workflow_run` workflow
for the privileged half. Platform changes that shift the defaults:

| Date | Change |
|------|--------|
| 2025-12-08 | `pull_request_target` always takes its workflow file and checkout ref from the **default branch**, not the PR's base branch, and environment branch rules are evaluated against the default branch — stale vulnerable workflows on old branches no longer run, but environment filters keyed to other branches need updating |
| 2026-06-18 | `actions/checkout` v7 refuses to check out fork PR head or merge code in `pull_request_target` workflows and in `workflow_run` workflows triggered by `pull_request*` events; the `allow-unsafe-pr-checkout` input is the explicit opt-out, and the same enforcement was backported to older majors except v1 |
| 2026-12-02 | On public repositories, `pull_request_target` is **blocked by default** unless an Actions event policy explicitly allows it (optionally per workflow file); blocked runs fail with an error annotation. Private and internal repositories and GHES are unaffected |

A workflow that genuinely needs `pull_request_target` (labeling, commenting on
forks) therefore needs an explicit policy allowance from 2026-12-02, must not
check out the PR head, and should run with `permissions:` reduced to the one
scope it writes.

## OIDC subject claims and cloud trust

Cloud trust policies should match the immutable numeric claims
(`repository_id`, `repository_owner_id`) rather than names alone: a deleted and
re-registered owner or repository name can otherwise mint tokens that satisfy a
name-based condition. GCP's Workload Identity Federation guide warns of exactly
this cybersquatting risk and requires an attribute condition for GitHub
providers. The numeric form also appears in environment subjects (see the OIDC
fact in the actions-facts reference above). The federation model itself is
owned by core's workload-identity reference.

## Provenance and immutable releases

- **Artifact attestations.** `actions/attest` (the successor that
  `actions/attest-build-provenance` v4 now wraps) signs SLSA provenance with
  Sigstore under the workflow's OIDC identity and needs `id-token: write` and
  `attestations: write`. On their own, attestations reach SLSA v1.0 Build
  Level 2; building in a reusable workflow shared across repositories provides
  the build isolation for Build Level 3. Consumers verify with
  `gh attestation verify <artifact> -R <owner>/<repo>` (or `oci://<image>` for
  container images) and should also assert the signer workflow.
- **Immutable releases (GA 2025-10-28).** With the setting on, a published
  release's assets and tag cannot be changed or deleted, and the release gets a
  signed attestation (`gh release verify <tag>`, `gh release verify-asset`).
  Publish as a draft, attach every asset, then publish — immutability starts at
  publication.

## Static analysis of workflows

Run both, as separate gates:

- **actionlint** (1.7.12 as of 2026-11-22) — syntax, expression typing,
  shellcheck over `run:` blocks, known-action input validation.
- **zizmor** (1.33.1 as of 2026-11-22) — security audit: template injection,
  `pull_request_target`/`workflow_run` hazards, excessive permissions,
  credential persistence, unpinned uses, cache poisoning. Its SARIF output can
  feed code scanning; CodeQL's `actions` language covers a similar class.

Pin both tools like any other dependency, and disclose any substitute linter
rather than claiming either ran.

## Sources

- https://docs.github.com/en/actions/reference/security/secure-use — SHA pinning, lockfile precedence, token permissions, script injection, `pull_request_target`/`workflow_run`, OIDC (accessed 2026-11-22)
- https://docs.github.com/en/actions/concepts/security/workflow-dependency-lockfiles — `actions.lock` contents, runtime verification, reusable-workflow exclusion (accessed 2026-11-22)
- https://github.blog/changelog/2026-10-28-github-actions-workflow-lockfiles-are-generally-available/ — lockfile GA, Require lockfile policy, Dependabot integration (accessed 2026-11-22)
- https://github.blog/changelog/2025-08-15-github-actions-policy-now-supports-blocking-and-sha-pinning-actions/ — SHA-pinning requirement and blocklist in the allowed-actions policy (accessed 2026-11-22)
- https://github.blog/changelog/2025-11-07-actions-pull_request_target-and-environment-branch-protections-changes/ — default-branch workflow source from 2025-12-08 (accessed 2026-11-22)
- https://github.blog/changelog/2026-06-18-safer-pull_request_target-defaults-for-github-actions-checkout — checkout v7 refusal and `allow-unsafe-pr-checkout` (accessed 2026-11-22)
- https://github.blog/changelog/2026-11-02-pull_request_target-blocked-by-default-december-2 — public-repository default block from 2026-12-02 (accessed 2026-11-22)
- https://docs.cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines — required attribute condition and numeric `*_id` claims for GitHub (accessed 2026-11-22)
- https://docs.github.com/en/actions/concepts/security/artifact-attestations — SLSA Build L2 by default, L3 via reusable workflows (accessed 2026-11-22)
- https://github.com/actions/attest-build-provenance — v4 wraps `actions/attest` (accessed 2026-11-22)
- https://github.blog/changelog/2025-10-28-immutable-releases-are-now-generally-available/ — immutable releases GA (accessed 2026-11-22)
- https://github.com/zizmorcore/zizmor and https://github.com/rhysd/actionlint — tool repositories; versions from their release tags (accessed 2026-11-22)
