# AGENTS.md — pipeline-workflows

The shared pipeline. Read this before changing a workflow here.

## What this repo is

One implementation of the CI/CD pipeline, called by project repos. A project's
own `.github/workflows/*.yml` is a thin caller: the trigger, the run name, the
permissions, the concurrency group, and its facts as `with:` inputs.

The project repos currently calling each workflow are listed in `README.md`.
That table is the source of truth — do not assume a workflow is shared because
it exists here.

## The invariants

1. **No project-specific literal in a workflow.** No hostname, repo name,
   directory or shell command that belongs to one project. If a workflow needs
   one, it is an input. This is the rule that keeps the repo reusable.
2. **No default for a project fact.** A missing host must fail loudly, not
   verify the wrong site. A silently skipped check is indistinguishable from a
   passing one.
3. **Project repos pin `uses:` to a full 40-character SHA.** Never a branch,
   never a tag. This repo's `main` is a moving target; a security boundary is not.
4. **A workflow change here is a policy change for every caller.** Test it
   against one project before bumping the others. The merge policy in
   particular decides what lands on `main` in every repo.
5. **The verdict semantics are not negotiable.** `error` means *could not judge*
   and never means *the code is wrong*. It blocks promotion all the same. See
   `docs/verify-contract.md`.

## Changing a workflow

```bash
actionlint -no-color .github/workflows/*.yml   # required; catches reusable-workflow mistakes
```

Then bump the pin in **one** project repo, watch a real run end to end, and only
then bump the rest. A YAML error in this repo reaches production as a skipped
gate, not as a red build — that is the failure mode to avoid.

## Verify before you believe

For `verify.yml`, a green run is not enough to call a change correct:

- the run must have posted `verify/<env>` on the sha it claimed to judge;
- `doctor` must have passed (identity), not just `journeys`;
- the verdict comment must name the count of journeys that ran.

A run that finishes green with no status posted reads exactly like a verified
artifact — the watchdog treats it as a broken gate (`alert_verify_unjudged`) for
that reason.

## House style

- Comments explain **why**, and name the incident that motivated a rule when
  there was one. The comments here are load-bearing: they are what stops a
  future edit from silently removing a condition.
- Fail closed. Refuse rather than guess. When something cannot be judged, say so
  in those words and use the `error` verdict.
- Shell in workflows stays POSIX-ish and `set -euo pipefail` (or `-uo` where a
  failing step must still report).
