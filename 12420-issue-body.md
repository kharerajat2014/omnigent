### What

`_run_installer` keeps a tail of a failed harness install's stderr so
`_installer_failure_reason` can name the cause. That tail is bounded by
**lines**:

```python
_INSTALLER_STDERR_TAIL_LINES = 80
...
tail: deque[str] = deque(maxlen=_INSTALLER_STDERR_TAIL_LINES)
```

A line has no length limit, so this bounds nothing. The comment above the
constant states the intent — "not the whole install log, which must not grow
the host daemon's memory" — but the implementation doesn't enforce it.

### Why it's reachable, not theoretical

Installers that draw a progress bar advance it with **carriage returns, not
newlines**. `for line in stream` doesn't yield until it sees a `\n`, so a
300-second install's entire progress output can arrive as one unterminated
"line" — which the deque then retains in full, inside the host daemon.

Line iteration also *accumulates* that output internally before yielding it,
so peak buffering is unbounded for the same reason. Truncating a line after
reading it would cap what is retained but not the peak.

Feeding 160 KB of `\r`-delimited progress followed by a real npm `EACCES`
block, comparing the current tail against a byte-bounded one:

```
old (line-keyed)     retained  163862 chars ( 10.0x the 16384 budget)  cause found: True
new (byte-bounded)   retained   16384 chars (  1.0x the 16384 budget)  cause found: True

installer emitted 163862 chars, all of it one unterminated line
```

Ten times over the intended budget from a single line, and 160 KB is just
what the reproduction emitted — the real ceiling is "however much the
installer writes before its first newline".

### Impact

Memory growth in the host daemon during a harness install, proportional to
installer output rather than to the fixed budget the code intends. No
incorrect behaviour otherwise: the cause is still diagnosed and the output is
still streamed.

### Where it came from

Raised as a non-blocking note in Polly's review of #12097, alongside two
others that are also still open: the installer test fake ignores the `Popen`
keyword arguments (so nothing asserts stderr stays piped and stdout stays
inheriting), and `try_install_harness_cli`'s docstring describes what the
reason used to be rather than the invariant. The blocking P1 from that same
review was fixed separately in #12185.

### Suggested fix

Read the pipe in bounded chunks and trim the tail to a byte budget, so both
retained bytes and peak buffering are bounded regardless of what the installer
emits. Chunked reads make a multi-byte character splittable across reads, so
the pass-through needs an incremental decode rather than a per-chunk one.

I have this written and tested and will open a PR against this issue.
