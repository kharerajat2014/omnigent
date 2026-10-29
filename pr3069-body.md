## Related issue

Closes #3069

Raised by @PattaraS in review of #2987 and filed by @xq-yin. I asked on the issue whether it was still wanted before opening this (it had been marked `stale` and was due to auto-close around 5 November); opening the PR so the work isn't lost to that clock.

## Summary

`try_install_harness_cli` ran the installer without capturing stderr, so when `npm install -g` failed the cause was never read and the only thing a UI-driven install could report was `installer exited with code 1`. An unwritable global directory and an unreachable registry are **both** exit code 1, which left them indistinguishable in the toast and gave the user nothing to act on.

This captures the installer's stderr and maps npm's own machine-readable `npm error code <CODE>` line to what the user can actually fix:

| npm code | reported as |
| --- | --- |
| `EACCES`, `EPERM` | permission denied writing the global install directory |
| `ECONNREFUSED`, `ECONNRESET`, `ENOTFOUND`, `ETIMEDOUT`, `EAI_AGAIN`, `ENETUNREACH` | could not reach the npm registry |
| `E401`, `E403`, `ENEEDAUTH` | registry rejected the credentials / refused access |
| `ENOSPC` | no space left on the device |

Matching the code rather than the prose keeps this stable across npm majors, and both spellings are handled (`npm error code …` on npm 10+, `npm ERR! code …` on 9 and earlier). The bare exit code remains the fallback, so a vendor install script — whose output isn't npm-shaped — still produces a reason.

### The part worth reviewing: reading the output must not stop it being displayed

The function's own docstring documented the old behaviour as deliberate:

> the installer's output streams to this process, uncaptured, so failures stay visible in the setup terminal / host log

So a plain `capture_output=True` would have fixed the toast while **regressing** that. With a 300 s install timeout, buffering everything to print at the end is also exactly the complaint in the sibling issue #3070.

So stderr is **teed**, not captured: piped, with each line written straight through to this process's stderr as it arrives, while a bounded tail is retained for the diagnosis. The tail is bounded because an install log must not grow the host daemon's memory. stdout is left inheriting and untouched. There's a test asserting the output still reaches this process, so the regression I was worried about can't creep back in.

### ELI5

A failed harness install used to say only "exited with code 1". Now it says *why* — "permission denied writing the global install directory (npm EACCES)" — while still printing the installer's own output as it goes.

## Test Plan

Reproduced first, against `6865742c6`, as two assertions:

1. The install subprocess was invoked with no `capture_output`, no `stderr=`, no `text=` — asserted on the kwargs. Nothing downstream *could* name the cause, because the cause was never read.
2. A real npm `EACCES` block and a real `ECONNREFUSED` block both produced the byte-identical string `installer exited with code 1`.

The throwaway repro was then converted into the committed tests and deleted.

**Tests added:**

- `test_try_install_names_the_npm_failure_cause` — parametrized over permission / unreachable registry / rejected credentials / no disk space, plus the npm 9 `npm ERR!` spelling.
- `test_try_install_falls_back_to_the_exit_code_for_unrecognized_output` — curl output from a vendor script still yields `installer exited with code 2`.
- `test_installer_stderr_still_reaches_this_process` — the tee guard described above.

**One thing reviewers should know about the test changes.** Reading stderr needs a pipe, so the install call is now a `Popen` rather than `subprocess.run`. The existing install tests stubbed `subprocess.run`, which meant that after this change they would **silently fall through to really running `npm install -g` or a vendor install script** on whatever machine ran the tests. That is not just a stale stub — I hit it, and it installed a real 2.4 GB vendor agent into my home directory and took the suite from 0.6 s to 5 m 46 s.

So rather than only re-pointing the stubs, the autouse fixture now replaces `subprocess.Popen` with one that fails loudly and names the helper to use instead, and `fake_installer()` builds the stand-in that exercises the tee. Any future test that forgets gets an assertion instead of a real install. Seven existing tests were re-pointed to it (five of mine-adjacent, plus `test_version_probe_refreshes_an_unchanged_launcher`, which is new on `main` and drives the install path).

**Runs:**

```
uv run --no-sync pytest tests/onboarding/test_harness_install.py \
  tests/host/test_connect.py \
  tests/server/integration/test_hosts_install_harness.py -q
# 605 passed, 1 skipped

uv run --no-sync pytest tests/onboarding -q \
  --ignore=tests/onboarding/sandboxes/test_databricks_sandbox.py \
  --ignore=tests/onboarding/sandboxes/test_databricks_sandbox_owner.py
# 1798 passed

uv run --no-sync pyrefly check omnigent/onboarding/harness_install.py
# 0 errors
```

`pre-commit run` passes on the staged diff. Rebased onto `afb637fed`; `git merge-tree` is conflict-free.

**Pre-existing failure, unrelated to this diff:** `tests/onboarding/sandboxes/test_databricks_sandbox{,_owner}.py` fail collection with `ModuleNotFoundError: No module named 'databricks.sdk.service.sandbox'` — an optional dependency missing from my environment. Verified identical on a stashed tree, hence the two `--ignore`s above.

## Demo

N/A — no UI change. The observable difference is the text of an existing toast, asserted directly in the tests.

## Type of change

- [x] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] UI / frontend change
- [ ] Documentation update
- [x] Refactor / chore

## Test coverage

- [x] Unit tests added or updated
- [ ] Integration / E2E tests added or updated
- [ ] Manual verification completed
- [ ] Not applicable

## Coverage notes

N/A — covered by unit tests, including a guard that the installer's output still reaches the terminal.

## Scope

Deliberately limited to the one-shot reason string, as #3069 scoped it. The fail-fast non-interactive flags and the elapsed-time hint are #3070, and streaming installer output to the UI needs a tunnel frame that both issues defer.

## Release notes

- [x] **Yes** — this change should appear in the release notes
- [ ] **No** — this change should not appear in the release notes

## Changelog

A harness CLI install that fails now reports why — permission denied on the global install directory, an unreachable npm registry, rejected registry credentials, or no disk space — instead of only `installer exited with code 1`. The installer's own output still streams to the setup terminal and host log as it runs.
