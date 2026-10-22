## Related issue

Closes #6700

## Summary

`github_policy`'s `write_repos` was a hard allowlist: a write to any repo outside
it was refused outright, with no way to route the decision to the user. That
leaves no room for a common shape — a trusted allowlist that should still permit
user-authorized work elsewhere. The alternatives were granting no writes at all,
or paying inference cost and latency for an LLM prompt policy to re-derive a
verdict this policy already parses deterministically.

Adds `write_violation_action`, defaulting to `DENY` so existing bundles are
unchanged. It follows `write_down_action` in the sibling `builtins/google.py`
rather than inventing a new convention: a string normalized and validated to
`DENY` or `ASK`, with a `ValueError` on anything else, plus a schema property so
it is reachable from a bundle. Both the MCP and git/gh shell surfaces read it from
the shared write gate, so the two cannot drift, and it covers a write whose target
could not be determined as well as one that misses the repo or branch allowlist.

**Destructive operations, force pushes and tag pushes keep denying under `ASK`.**
Those are refused on their own merits rather than for missing the allowlist, and
this is the one part that is not a one-line change. The MCP path consults its
destructive guard *after* the allowlist gate — deliberately, so a delete on a
non-allowed repo reports the repo denial. Escalating naively turns that ordering
into a bypass: the allowlist miss returns `ASK` and returns before the destructive
guard is ever reached, so `delete_branch` on an unlisted repo would prompt instead
of refuse. The gate now refuses to soften a violation for an op that is
independently denied, which also keeps every existing denial reason
byte-identical in the default configuration.

Not a duplicate of #765, which the triage bot flagged as possibly related. That
issue is about a mid-flight `ASK` not reaching the user on a native session at the
later phases. This policy decides at `tool_call`, where `ASK` is honoured as a
pending elicitation (`runner/policy.py`), and where the existing policy already
returns `ASK` today for shell commands with unresolvable targets.

## Test Plan

Reproduced the gap before fixing. A throwaway test asserted that the factory
raised `TypeError` on `write_violation_action` and that `validate_factory_params`
rejected it, proving the behaviour was unreachable from both Python and a bundle,
alongside the hard `DENY` for an unlisted repo and an undeterminable target. All
five assertions passed against unmodified `main`; the first two now fail by
construction, and the rest were converted into the regression tests here.

```
uv run --no-sync pytest tests/policies -q
# 770 passed

uv run --no-sync pytest tests/policies/builtins/test_github.py -q
# 232 passed (211 before — 21 new)

uv run --no-sync pyrefly check omnigent/policies/builtins/github.py
# 0 errors

uv run --no-sync pre-commit run --files docs/POLICIES.md \
  omnigent/policies/builtins/github.py tests/policies/builtins/test_github.py
# ruff format, ruff check, pyrefly + all hooks: Passed
```

The first commit is independently green on the pre-existing suite (211 passed)
without any test changes, so the behavioural default is provably untouched.

Manual verification:

- With `write_repos: [myorg/trusted]` and `write_violation_action: ASK`, asking an
  agent to open a PR against a different repo produces an approval prompt naming
  the repo rather than a refusal that ends the turn. Removing the setting restores
  today's refusal.
- Still in `ASK`, asking it to delete a branch on that same unlisted repo refuses
  — the guard the ordering fix protects.
- `write_violation_action: warn` fails loudly at construction with a message
  naming the parameter, instead of booting with the gate silently widened.

## Demo

- [ ] Visual demo attached below
- [x] Non-visual evidence provided below or in Test Plan
- [ ] Not applicable — no behavioral change

Non-visual change (a policy configuration surface), so here is the resolved
verdict for every operation shape under both settings:

```
operation                         DENY (default)      ASK
----------------------------------------------------------
write allowlisted repo                     ALLOW    ALLOW
write unlisted repo                         DENY      ASK
write undeterminable target                 DENY      ASK
destructive on unlisted repo                DENY     DENY
shell push to unlisted repo                 DENY      ASK
shell force push                            DENY     DENY
shell tag push                              DENY     DENY
```

The `DENY` column is byte-identical to `main`, which is the
backward-compatibility claim. The three `DENY DENY` rows are the guards that must
survive escalation.

And the rejection of an unusable value:

```
'warn'     -> ValueError: github_policy: write_violation_action must be 'DENY' or 'ASK', got 'warn'
'ALLOW'    -> ValueError: github_policy: write_violation_action must be 'DENY' or 'ASK', got 'ALLOW'
''         -> ValueError: github_policy: write_violation_action must be 'DENY' or 'ASK', got ''
```

## Type of change

- [ ] Bug fix
- [x] Feature
- [ ] UI / frontend change
- [ ] Refactor / chore
- [ ] Docs
- [ ] Test / CI
- [ ] Breaking change

## Test coverage

- [x] Unit tests added / updated
- [ ] Integration tests added / updated
- [ ] E2E tests added / updated
- [x] Manual verification completed
- [x] Existing tests cover this change
- [ ] Not applicable

## Coverage notes

21 new unit tests across the file's existing layers. They cover the `DENY`
default, escalation for an unlisted repo, an unlisted branch, an undeterminable
target and the shell surface, the YAML spellings an operator is likely to write
(`ask`, `Ask`), and a `ValueError` for each unusable value.

The load-bearing ones are the guards that must survive `ASK`: a destructive op on
an unlisted repo, a destructive op naming no repo at all, force pushes and tag
pushes. Those are precisely what a naive escalation opens up, and each fails
against an implementation that only swaps the verdict at the allowlist check.

At the registry layer, one test pins the published enum and default so the setting
is discoverable, and asserts a wrong type is rejected. Worth noting for reviewers:
`validate_factory_params` is type-only and does not enforce enum, so a bad string
passes schema validation and is caught by the factory's `ValueError` instead. That
matches the sibling `gdrive_policy` exactly; the test documents the split rather
than implying the schema is the gate.

Manual verification covered what unit tests cannot: that the `ASK` verdict
actually surfaces as an interactive approval in a live session rather than being
collapsed, and that a typo fails at construction instead of at some later write.

## Release notes

- [ ] No — no noteworthy user-facing change.
- [x] Yes — include the entry in the Changelog section.

## Changelog

`github_policy` accepts `write_violation_action: ASK`, so an agent with a trusted
repo allowlist can ask for approval on writes elsewhere instead of failing the
turn.
