# CI Workflow Conventions

CI workflow conventions — path-filtered triggers, fail-loudly posture, rationale comments, managed-file materialization, and run-scoped artifact directories.

Workflows are production code with worse review habits. These conventions keep a
growing workflow tree cheap to reason about: each workflow runs when its inputs
change, fails loudly rather than papering over problems, explains itself, and
produces artifacts that stay attributable.

## Path-filtered triggers derived from outputs

A workflow should run when something it produces could have changed — not on every
push. Derive the filter from the workflow's outputs, in that direction:

1. List what the workflow produces (a compiled binary, a container image, a lint
   verdict, a documentation build).
2. List the inputs that can change each output (source trees, build files, the
   workflow itself, dependency manifests).
3. Write the path filter from that map, and include the workflow's own path so
   trigger changes are always exercised.

This derivation matters because convention-written filters rot: they accumulate
paths nobody remembers justifying and omit inputs added later. Re-derive when a
workflow's outputs change. Monorepos feel this most — without output-derived
filters, every commit runs everything.

## Fail-loudly baseline posture

The default CI posture is to fail on ambiguity. Concretely:

- Auto-update and auto-fix flags stay off in CI (snapshot updates, lockfile
  regeneration, formatter writes). CI verifies the committed state; if generated
  output is stale, the run fails with a diff, and a human runs the generator and
  reviews the result locally. A CI that "fixes" drift silently turns review into
  theater.
- Set strict modes (shell error propagation, unset-variable errors, glob-null
  behavior) at the top of script steps.
- Failures name the failing field or step; a red run whose only signal is a
  non-zero exit teaches nobody anything.

## Rationale-commented workflows

Every non-obvious step carries a comment stating why it exists — the incident,
constraint, or external requirement it serves. What a step does is readable from
the step; why it does it is not, and the why is what prevents the next well-meaning
cleanup from deleting a load-bearing workaround. A step whose why cannot be stated
is a candidate for deletion.

## Managed-file materialization for fresh clones

Anything derived — generated code, managed configuration, vendored schemas — must
be materialized by an explicit workflow step before anything consumes it. The test
of this convention: a fresh clone plus one pipeline run builds successfully with no
undocumented local setup. If a workflow only passes on workstations where a
developer previously ran some script by hand, the materialization step is missing
or mis-ordered, and the next fresh environment (a new hire, a runner image bump, an
outage recovery) will find it.

## Run-scoped artifact directories

Write run outputs — logs, reports, coverage, build exports — under a path scoped to
the run (run id, commit sha, or both), never to a shared fixed path. Parallel runs
on the same branch must not overwrite each other, and any artifact must remain
attributable to the exact run that produced it. Fixed shared paths are the storage
equivalent of a mutable tag: convenient, and quietly wrong under concurrency.
