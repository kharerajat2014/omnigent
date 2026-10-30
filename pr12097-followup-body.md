## Related issue

Follow-up to #12097 (which closed #3069), addressing the non-blocking notes
from Polly's review there. The blocking P1 from that same review — cleanup
hanging past the install timeout when a child holds stderr — was already fixed
by @PattaraS in #12185; this builds on that and does not revisit it.

Three notes were left open. This takes the two substantive ones and the
wording one:

| Review note | Status before this PR |
| --- | --- |
| The retained tail is bounded by lines, not memory | `deque(maxlen=80)` over line iteration |
| The subprocess-boundary assertion is weak | `FakeInstaller` ignored the `Popen` kwargs |
| The changed historical explanation should state the invariant | docstring still said "were previously reported identically" |

## Summary

**The tail bounded nothing.** `_INSTALLER_STDERR_TAIL_LINES = 80` reads like a
memory bound, but a line has no length limit. The case that makes this real
rather than theoretical: an installer drawing a progress bar emits **carriage
returns, not newlines**, so a 300-second install's entire run of output can
arrive as one unterminated "line" — which the tail then held in full, in the
host daemon.

Measured on the shipped implementation versus this one, feeding 160 KB of
`\r`-delimited progress followed by a real npm `EACCES` block:

```
old (line-keyed)     retained  163862 chars ( 10.0x the 16384 budget)  cause found: True
new (byte-bounded)   retained   16384 chars (  1.0x the 16384 budget)  cause found: True

installer emitted 163862 chars, all of it one unterminated line
```

10x over budget from a single line, and that is just 160 KB — the real bound
was "however much the installer emits".

**Why not just truncate the line.** Polly called this out and it's correct:
line iteration *accumulates* newline-free output before yielding it, so
trimming a completed line bounds what is retained but not the peak. The fix
has to move off line iteration.

So the pipe is read in bounded chunks and the tail is trimmed to a byte
budget. Retained bytes and peak buffering are now both bounded no matter what
the installer emits — 16 KiB of tail, plus at most one 4 KiB read in flight.

**Two consequences worth reviewing.**

1. Reading bytes means a chunk can end mid-character, so the pass-through
   decodes **incrementally** rather than per chunk; a per-chunk decode would
   mangle any multi-byte character landing on a boundary. There's a test that
   puts one exactly astride the first boundary.
2. Bounded reads need `read1` — the read that returns what has arrived instead
   of blocking until it has a full chunk, which a plain `read(n)` would do and
   which would stall the live output the tee exists to preserve. `Popen` types
   `stderr` as `IO[bytes]`, which doesn't declare `read1`, hence the narrow
   structural type (same shape as `_TermUIWithHiddenPrompt` in
   `onboarding/interactive.py`).

The tail is still only as long as it needs to hold npm's error block, and the
reason strings are unchanged — this is purely about what the daemon retains
while producing them.

### ELI5

The code kept "the last 80 lines" of a failed install's output to work out
why it failed. But some installers print progress without ever starting a new
line, so "80 lines" could mean the entire install log sitting in memory. Now
it keeps the last 16 KB instead, which can't grow.

## Test Plan

**The bound, reproduced first.** The old-versus-new comparison above is a
standalone script running both tail implementations over the same
newline-free input; it is the evidence for the numbers quoted. It was a
throwaway and is not committed — the committed form is the assertion below.

**Tests added:**

* `test_installer_tail_stays_bounded_against_newline_free_output` — output
  that never terminates a line must not grow the retained tail
  (`len(tail) <= _INSTALLER_STDERR_TAIL_BYTES`), the cause must still be
  diagnosable from what is kept, and the output must still be displayed in
  full. That last assertion is the one that would catch "bounded the tail by
  truncating the tee".
* `test_installer_tee_decodes_a_character_split_across_chunks` — a multi-byte
  character placed astride the first chunk boundary comes through intact.

**Test helper change reviewers should know about.** `FakeInstaller` ignored
the `Popen` keyword arguments, so every install test passed whether or not
stderr was still piped and stdout still inheriting — the boundary the tee
depends on. It now asserts them, which also pins the absence of text mode
that chunked reads require. This is what makes the fake's stand-in credible
rather than just convenient; it applies to all ~14 install tests, not only the
new ones.

The two real-subprocess tests @PattaraS added in #12185 are unchanged and
still pass, which is the check that matters most here: they exercise the
actual pipe, the kill-tree path and the bounded drain against real child
processes, none of which the fake can prove.

**Runs:**

```
.venv/bin/python -m pytest tests/onboarding/test_harness_install.py \
  tests/host/test_connect.py \
  tests/server/integration/test_hosts_install_harness.py -q
# 617 passed, 1 skipped

.venv/bin/python -m pytest tests/onboarding -q \
  --ignore=tests/onboarding/sandboxes/test_databricks_sandbox.py \
  --ignore=tests/onboarding/sandboxes/test_databricks_sandbox_owner.py
# 1865 passed

.venv/bin/python -m pyrefly check omnigent/onboarding/harness_install.py
# 0 errors
```

`pre-commit run --from-ref upstream/main --to-ref HEAD` passes. Branched from
`ef7d57914`, so #12185 is included.

**Note on the invocation:** `uv run` was not used because `[tool.uv]` now
requires `uv >= 0.13.4` and this machine has 0.12.5; the project venv was
driven directly instead. Same interpreter, same installed deps.

**Pre-existing failure, unrelated to this diff:**
`tests/onboarding/sandboxes/test_databricks_sandbox{,_owner}.py` fail
collection with `ModuleNotFoundError: No module named
'databricks.sdk.service.sandbox'` — an optional dependency missing from my
environment, hence the two `--ignore`s. Identical on an unmodified tree.

## Demo

N/A — no user-visible change. The reason strings and the streamed output are
byte-for-byte the same; what changes is the daemon's memory ceiling while
producing them.

## Type of change

- [x] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] UI / frontend change
- [ ] Documentation update
- [ ] Refactor / chore

## Test coverage

- [x] Unit tests added or updated
- [ ] Integration / E2E tests added or updated
- [ ] Manual verification completed
- [ ] Not applicable

## Coverage notes

N/A — covered by unit tests, including the display-still-intact assertion that
guards against bounding the tail by truncating the tee.

## Scope

Limited to the three open review notes on #12097. The sibling issue #3070
(fail-fast flags and an elapsed-time hint for a stuck install) is untouched,
as is the UI streaming both issues defer.

## Release notes

- [ ] **Yes** — this change should appear in the release notes
- [x] **No** — this change should not appear in the release notes
