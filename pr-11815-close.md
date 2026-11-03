Closing this as superseded by #12517, which landed the same capability with a broader design.

`main` now keeps a per-harness table of launchable permission modes in `omnigent/native/permission_modes.py`, shared by the persist-time check and the fire path, which is the shape this PR proposed. For `codex-native` it accepts `bypass`, which `fire.py` turns into the same bypass-sandbox label New Chat uses, and `full-access` (`--sandbox danger-full-access --ask-for-approval never`). Either lets a scheduled codex task run unattended instead of parking on a file-change approval.

Rebasing this would mean reintroducing a second vocabulary (`bypassPermissions` for codex) next to the one #12517 settled on, so there's nothing left here worth landing. Thanks to @CSteigstra for #6272, which first worked this out.
