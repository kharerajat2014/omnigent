## Related issue

Closes #5981

Follow-up to #5980, which added the Kubernetes-specific `pod_ready_timeout_s`
override and explicitly deferred this provider-agnostic gap.

## Summary

`MANAGED_HOST_ONLINE_TIMEOUT_S` was a hardcoded 120 with no config key and no
env var. It is the budget every managed launch waits for the host process inside
the sandbox to register itself — the wait that begins after a provider's own "is
my instance running" wait (Kubernetes's `pod_ready_timeout_s`, for example)
already succeeded. A deployment whose host image or first-boot script is slower
than 120s could only raise it by patching the constant.

Adds `OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S`, resolved at call time by
`resolve_managed_host_online_timeout_s()`, following the pattern
`OMNIGENT_E2B_MAX_LIFETIME_S` already uses in `sandboxes/e2b.py`. Env var only,
no config key: the constant lives in `managed_hosts.py` rather than any one
launcher, so there is no per-provider block for it to sit in the way
`pod_ready_timeout_s` has. A malformed value raises a clear `ClickException`, and
so does a zero or negative one — 0 parses fine as a float but would fail every
launch before a healthy host could register.

**Fixes a second defect the override exposes.**
`MANAGED_LAUNCH_RENDEZVOUS_TIMEOUT_S` was derived from the constant at import
time (+ 120 of slack) and bounds how long a parked message POST waits for an
in-flight launch to settle — by design it must outlive the registration wait.
Frozen at import, raising the registration budget to 600s left the rendezvous at
240s, so the launch would patiently wait 600s while the parked message gave up at
240s, losing the first post-dormancy turn on exactly the slow launches the
override exists to support. Both budgets now come from the same resolver via
`managed_launch_rendezvous_timeout_s()`.

Renames the constant to `DEFAULT_MANAGED_HOST_ONLINE_TIMEOUT_S` so nothing keeps
reading a value that now ignores the operator's override. That rename is what
surfaced the eight tests monkeypatching it, which now drive the env var instead
and so exercise the new path rather than bypassing it.

Documents the variable in `docs/extending/sandbox_providers.md` (it is
provider-agnostic, so it belongs in the shared reference) and points the CW
Sandbox and Daytona "slow first launch" bullets at it. Deliberately left the Islo
and Daytona "did not come online" troubleshooting bullets alone: those attribute
the failure to egress firewalls blocking the host's dial-back, and suggesting a
longer timeout there would send operators down the wrong path.

## Test Plan

Reproduced before fixing. A throwaway test set the env var to `0.05` and asserted
a launch whose host never registers fails fast; against unmodified `main` it
instead kept polling toward the hardcoded 120s and the bounded 3s wait raised
`TimeoutError`. A second check confirmed the rendezvous budget stayed at 240s no
matter what the env var said. Both were then converted into the regression tests
in this PR.

```
uv run --no-sync pytest tests/server/test_managed_hosts.py \
  tests/server/integration/test_host_session_binding.py -q
# 283 passed

uv run --no-sync pre-commit run --from-ref main --to-ref HEAD
# ruff format, ruff check, pyrefly + all hooks: Passed
```

Manual verification:

- `OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S=abc` and `=0` are both refused with a
  clear message naming the variable, not a traceback.
- With the override at `5` and a host that cannot dial back, the 502 reads
  `managed host did not come online within 5s` and arrives in about five seconds
  instead of two minutes — confirming both that the override reaches the wait and
  that the message quotes the resolved budget rather than always claiming 120s.
- Unset, behaviour is unchanged at 120s.

## Demo

Non-visual change (server-side config surface), so here is the observable
behaviour instead. Both budgets resolved together, across every input shape:

```
(unset)                                              online=120s   rendezvous=240s
OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S=600           online=600s   rendezvous=720s
OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S=0.05          online=0.05s  rendezvous=120.05s
OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S=abc           ClickException: OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S must be a number of seconds
OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S=0             ClickException: OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S must be greater than zero
OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S=-30           ClickException: OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S must be greater than zero
```

The rendezvous column is the point of the second commit: on `main` it stays at
240s for every row, which is below the 600s wait it is supposed to outlive.

And the launch-failure message now quoting the resolved budget rather than always
saying 120s:

```
managed host did not come online within 0.05s — check /tmp/omnigent-host.log inside the sandbox
```

- [ ] Visual demo attached below
- [x] Non-visual evidence provided below or in Test Plan
- [ ] Not applicable — no behavioral change

## Type of change

- [x] Bug fix
- [x] Feature
- [ ] UI / frontend change
- [ ] Refactor / chore
- [ ] Docs
- [ ] Test / CI
- [ ] Breaking change

## Test coverage

- [x] Unit tests added / updated
- [x] Integration tests added / updated
- [ ] E2E tests added / updated
- [x] Manual verification completed
- [x] Existing tests cover this change
- [ ] Not applicable

## Coverage notes

Unit tests cover the resolver directly: the 120s default when unset, an honoured
override, and a `ClickException` for each shape of unusable value (non-numeric,
empty, 0, negative). The rendezvous test is the regression guard that matters
most — it asserts the parked-message budget both tracks a raised override and
stays strictly above it, which is what the import-time derivation silently got
wrong. The existing online-timeout test now asserts the resolved budget appears
in the 502 detail.

The seven integration patches in `test_host_session_binding.py` and the one in
`test_managed_hosts.py` were switched from monkeypatching the module constant to
setting the env var, so the existing managed-launch suite now exercises the new
resolution path end to end.

Manual verification covered what tests cannot: that an operator setting a bad
value sees a clean CLI error rather than a traceback, and that a real launch
against an unreachable `server_url` fails at the configured budget with the
resolved number in the message.

## Release notes

- [ ] No — no noteworthy user-facing change.
- [x] Yes — include the entry in the Changelog section.

## Changelog

Managed sandbox launches honour `OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S`, so
deployments with slow-booting host images can raise the host-registration budget
without patching the source.
