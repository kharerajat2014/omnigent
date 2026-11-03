This looks resolved on `main` by #12517: a scheduled `codex-native` automation can now save the `bypass` or `full-access` permission mode, so it no longer parks on a file-change approval with nobody to answer it. It should ship in the next release after v0.20.0.

@CSteigstra, if you still see a scheduled codex task waiting for approval after upgrading, could you say which version and permission setting you're using? Otherwise this can be closed.
