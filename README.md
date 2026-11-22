# pipeline-workflows

The shared implementation of the issue → spec → PR → CI → merge → deploy →
verify pipeline. **One copy of the logic, called by every project repo.**

Public on purpose: it holds no secrets (deploy keys stay in the project repos and
are passed down with `secrets: inherit`), and a SHA-pinned public file is
auditable by anyone. It is also out of every agent's reach — the `pm-*` and
`coder` GitHub tokens are scoped to a single project repo, so an agent cannot
change the policy that judges it.

## State: what has moved, and what has not

| Workflow | Lives here | Called by |
|---|---|---|
| `verify.yml` | ✅ yes | `rudijs/ed-app-1` |
| `merge.yml` · `deploy.yml` · `promote.yml` · `rollback.yml` · `ci.yml` | ❌ not yet | still owned by each project repo |

This table is the source of truth for the migration. Do not claim a workflow is
shared until its row says so, and do not delete a project repo's copy until its
row does.

The migration order is **merge → deploy → promote → rollback → ci**, because that
is the order in which drift costs the most: the merge policy is what turns "CI
green" into "code on main", so two diverging copies of it are two different
security postures.

## Why this repo exists

The project repos had ~112 KB of near-identical YAML, and the copies had already
diverged in ways that changed behaviour (issue closure on verify vs on merge; a
promotion gate that requires `verify/dev` in one repo and not the other). Drift
between copies is not a tidiness problem — it means the invariant "production
never runs an unverified artifact" is true in one repo and false in another.

A copy that is called cannot drift. That is the whole design.

## The verify contract

Every project's verification must produce four things for a deployed artifact:

1. **Identity** — the served artifact is the commit under test.
2. **Behaviour** — at least one assertion about what the artifact *does*.
3. **Verdict** — `verify/<env>` on that sha: `success` | `failure` | `error`.
   `error` means *could not judge* (unreachable, TLS, wrong build, runner
   problem) and never means *the code is wrong*. It blocks promotion all the same.
4. **Evidence** — something a human can read without a runner: a log tail, a
   screenshot pair, an artifact.

Fuller version: [`docs/verify-contract.md`](docs/verify-contract.md).

## Adopting it in a project

1. Copy [`templates/verify-caller.yml`](templates/verify-caller.yml) into
   `.github/workflows/verify.yml` in the project repo.
2. Fill in the project facts: `host_dev`, `host_prod`, `evidence_dir`, and the
   commands (`install_command`, `doctor_command`, `journeys_command`,
   `shots_command`).
3. Pin `uses:` to a **full 40-character commit SHA**, never a branch or a tag.
   The policy is a security boundary; a moving ref is a moving boundary.
4. Make sure the project has the commands the caller names. A missing one fails
   the run loudly — a silently skipped gate is indistinguishable from a pass.

The project repo keeps the trigger, `run-name`, `permissions` and `concurrency`.
This repo owns everything after that. If you find yourself adding a project name,
a host, a path or a command to a workflow in *this* repo, it is in the wrong
repo — add an input instead.

## Bumping the pin

Deliberate, reviewed, and one PR:

```bash
# in the project repo
sed -i 's|pipeline-workflows/.github/workflows/verify.yml@[0-9a-f]\{40\}|pipeline-workflows/.github/workflows/verify.yml@<new-sha>|' \
  .github/workflows/verify.yml
```

A workflow change here is a policy change for **every** repo that calls it. Test
it against one project before bumping the rest.

## Validating a change

`lint.yml` runs `actionlint` on every push and pull request that touches a
workflow — the gate for the gates, because a YAML mistake here becomes a
silently skipped step downstream, not a red build. Locally:

```bash
actionlint -no-color .github/workflows/*.yml
```

`actionlint` is a single binary; `~/.local/bin/actionlint`. It catches reusable
workflow mistakes (undeclared inputs, `workflow_run` misuse, expression errors)
that GitHub would otherwise only reveal at run time, on a live pipeline.
