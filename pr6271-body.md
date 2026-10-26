## Related issue

Closes #6271

**Credit:** @CSteigstra's #6272 implemented this first and was reviewed green before being auto-closed under `waiting-on-author` — the resolve agent's final round reported "150 passed" with "`Maintainer Approval` is the pending human gate." I arrived at the same design independently, then read #6272's description and review history and took its file list as my checklist. I did not copy its diff. I offered on #6271 to stand down if @CSteigstra wanted to `/reopen` (their branch still exists); there was no reply, so this re-lands the same shape.

## Summary

A scheduled `codex-native` task could not run unattended. Every fire parked on a `codex_file_change_approval` elicitation with nobody there to answer it, and there was no way to opt out:

- `validate_permission_mode_agent_support` allowed a `permission_mode` only on `claude-native`, so the bypass opt-in was refused at persist time with a 400.
- `fire._permission_mode_launch_args` knew only Claude's `--permission-mode` translation, so even a hand-written database row produced no flag.

This replaces the single-harness check with a per-harness table of launchable modes plus one shared `permission_mode_launch_args()` mapper, used by **both** the persist-time gate and the fire path so the two can't drift — whatever the mapper can't translate is exactly what the gate rejects.

```text
scheduled task permission_mode
  |
  +-- claude-native : full vocabulary -> ["--permission-mode", <mode>]
  +-- codex-native  : "bypassPermissions" only
  |                   -> ["--dangerously-bypass-approvals-and-sandbox"]
  `-- unset / unsupported pair / unresolved harness
                      -> no args, keeps prompting (unchanged)
```

Graded Claude modes (`acceptEdits`, `plan`, …) stay rejected for codex rather than accepted and ignored: codex has no equivalent, and silently dropping the mode would leave an unattended task prompting with nothing to say so.

### Why `permission_mode` rather than a new field

This isn't inventing a cross-harness meaning for Claude's vocabulary. `_derive_terminal_launch_args_from_spec` already reads `permission_mode: bypassPermissions` as the spelling for full bypass on **antigravity-native**, mapping it to that CLI's `--dangerously-skip-permissions`. "Explicit `bypassPermissions` means full bypass, translated to each harness's own flag" is the established seam, so this needs no new field, no schema migration, and no change to the wire format.

`terminal_launch_args` is also already how a headless codex worker receives this exact flag, so the fire path needed no new transport — only the mapping.

### ELI5

A scheduled Codex run used to stop and wait for a human to approve a file edit, which never came, so the run did nothing. You can now explicitly say "this one runs without asking." Say nothing and behaviour is exactly as before.

## Security framing

This widens a permission gate, so to be explicit about the boundaries:

- **Opt-in only.** Unset behaviour is byte-identical; nothing becomes more permissive by default.
- **One mode, not a family.** Codex admits `bypassPermissions` and nothing else.
- **The fail-safe is preserved and tested.** An unresolvable harness, or a harness/mode pair with no mapping, yields *no* args — so a mis-stamped row can neither break a launch with an unknown flag nor silently inherit a bypass it never asked for. That's covered by a parametrized test including `cursor-native`, which has no mapping at all.
- **Still gated at persist time**, so a caller gets a 400 rather than discovering at fire time that its mode was dropped.
- The docs now say plainly what the mode grants: "`bypassPermissions` gives an unsupervised run full authority in its workspace."

## Scope

Deliberately **codex only**, matching the issue. Scheduled `antigravity-native` tasks have the same gap (the spec seam already maps the mode for interactive launches), but widening a second bypass in a PR nobody asked for felt wrong — happy to file that separately, or to fold it in here if a maintainer prefers.

No UI change: the create dialog still doesn't offer the control for codex, so this reaches users through the REST API and the `sys_scheduled_task_*` tools. Same scope #6272 was reviewed at.

## Test Plan

Reproduced first, against unmodified `main` at `6865742c6`, as three separate assertions:

1. `validate_session_permission_mode("bypassPermissions")` **passes** — the value was already a legal persisted mode.
2. `validate_permission_mode_agent_support(..., agent=<codex-native>)` raised `permission_mode is only supported for claude-native agents, not 'codex-native'` — the reported 400.
3. Bypassing that gate, `_permission_mode_launch_args` returned `None` for a codex task carrying the mode — so the flag was dropped even if persisted.

A control confirmed claude-native still produced `["--permission-mode", "bypassPermissions"]`. The throwaway repro was then converted into the committed tests below and deleted.

**Tests added / changed:**

- `tests/server/routes/test_session_create_validation.py` — a parametrized unit test of the new shared mapper across all seven branches: both Claude cases, the codex bypass, a Claude-only mode on codex, an unmapped harness, `harness=None`, and `permission_mode=None`.
- `tests/server/scheduled/test_fire.py` — new `test_codex_bypass_permissions_becomes_the_codex_bypass_flag` asserting the real fire produces the codex flag. **Rewrote** `test_permission_mode_omitted_for_non_claude_agent`, which asserted the old behaviour (codex + `bypassPermissions` → no args); it is now `test_permission_mode_omitted_for_unsupported_harness_mode`, parametrized over `codex-native`+`acceptEdits`, `codex-native`+`plan`, and `cursor-native`+`bypassPermissions`, so the fail-safe guarantee it was protecting is still pinned — on pairs that genuinely have no mapping.
- `tests/server/integration/test_scheduled_tasks_routes.py` — new `test_create_persists_bypass_permission_mode_for_codex_agent` driving the real `POST /v1/scheduled-tasks` and reading the row back. Renamed the existing rejection test to `..._rejects_claude_only_permission_mode_for_codex_agent` and corrected its docstring, since "non-Claude agent" is no longer why it's rejected.

**Runs:**

```
uv run --no-sync pytest tests/server/scheduled \
  tests/server/routes/test_session_create_validation.py \
  tests/server/integration/test_scheduled_tasks_routes.py \
  tests/runner/test_scheduled_task_tool_dispatch.py -q
# 449 passed
uv run --no-sync pyrefly check omnigent/server/routes/_session_create_validation.py \
  omnigent/server/scheduled/fire.py
# 0 errors
```

`git merge-tree` against current `upstream/main` is conflict-free.

**Pre-existing failures, unrelated to this diff** (both reproduce on unmodified `main` — verified by stashing):

- `ruff check` reports `BLE001` at `omnigent/server/scheduled/fire.py:937`, in a cost-budget `except` block this PR doesn't touch.
- `web TypeScript type check` fails in `TipTapCodeBlockExtension.ts`, `tiptapMarkdownPatches.ts`, `TipTapNodeViewSelection.test.tsx`, `TipTapRendererRegistry.test.tsx`, and `test-setup.ts`. None are files this PR touches, and both web edits here are doc comments only.

## Demo

N/A — no UI change. The behaviour is a launch flag on an unattended run; the observable difference is a scheduled codex task completing instead of stalling, which the tests assert directly.

## Type of change

- [x] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] UI / frontend change
- [x] Documentation update
- [ ] Refactor / chore

## Test coverage

- [x] Unit tests added or updated
- [x] Integration / E2E tests added or updated
- [ ] Manual verification completed
- [ ] Not applicable

## Coverage notes

N/A — covered by automated tests at three layers (the mapper, the fire path, and the REST API).

## Release notes

- [x] **Yes** — this change should appear in the release notes
- [ ] **No** — this change should not appear in the release notes

## Changelog

Scheduled Codex automations can now opt into `permission_mode: "bypassPermissions"`, which starts the run with Codex's `--dangerously-bypass-approvals-and-sandbox` stance instead of stalling on a file-change approval that an unattended run has nobody to answer. Behaviour is unchanged when the field is unset, and Claude-only modes such as `acceptEdits` remain rejected for Codex agents.
