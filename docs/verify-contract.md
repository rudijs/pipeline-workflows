# How verification is judged

`verify.yml` is the phase that answers **"does the deployed artifact work?"**
Deploy answers "it shipped". This is the one that decides whether a change is
*verified* — and "verified" is what lets a merged change close its issue, and
what gates promotion to production.

Three verdicts, on the commit under test, posted as `verify/<env>`:

| state | means | who owns the fix |
|---|---|---|
| `success` | probe + doctor + every journey passed | — |
| `failure` | the artifact is live and does something wrong | the pipeline (re-delegate) |
| `error` | **could not judge**: unreachable, TLS broken, stale build on the host, runner could not install a browser, a configured URL disagrees with the deployment record | the operator |

`error` is not a softer `failure`; it is a different owner. Collapsing them sends
someone hunting for a code defect in a healthy app. That distinction is the
single most important property of this workflow, and it exists because it was
got wrong on 2026-11-19 (a configured-URL override produced a probe failure that
read like a bug).

## The four things a verification must produce

Every project that adopts this workflow produces these, whatever its stack:

1. **Identity** — proof that the artifact being driven is the commit under test,
   not a cached page or a stale container. Web: a build id stamped at build time
   and read back from the served page. CLI: `--version` output. Mobile: the
   build number in the installed app. This is the `doctor` step.
2. **Behaviour** — at least one assertion about what the artifact does, driven
   the way a user drives it. This is the `journeys` step, and it is **the only
   step whose failure means the artifact is wrong**.
3. **Verdict** — one commit status per environment, machine-readable, so the
   promotion gate can read it without parsing prose.
4. **Evidence** — a log tail, a before/after screenshot pair, or an artifact a
   human can open without a runner.

## Why the suite compounds

`verify` checks out the commit under test and runs *that commit's* suite against
the *deployed* environment. So a journey added for one feature keeps judging
every later deploy. Trust here does not come from the workflow being clever; it
comes from the suite growing with the product. A feature that ships without a
journey is a feature nothing will ever re-check.

That is why the spec should name the check (`pm-verify`) and the merge policy
should refuse a behavioural change that adds no test: the pipeline's ability to
say "verified" is exactly the size of its suite.

## Front-load the cheap checks

Order matters, and it is deliberate:

1. **probe** — is anything there, with valid TLS? (10 attempts, ~50s)
2. **doctor** — is it the right build on the right host?
3. **journeys** — does it do the thing?

A dead route is not a code defect, and there is no point driving a browser at a
stale build. Each step's failure produces a verdict with the right owner.

`install` failures are inconclusive too: a runner that cannot install a browser
says nothing about the deployed artifact.

## What is intentionally not here

- **No retry of the whole suite.** At most one retry per test (`retries: 1`), so
  flake stays visible in the log instead of being averaged away.
- **No verification inside deploy.** Re-verifying must not redeploy, and a
  pull request must not be able to change the gate that judges it.
- **No promotion decision.** This workflow posts the verdict; the project's
  promotion gate reads it. Production stays a deliberate act.
