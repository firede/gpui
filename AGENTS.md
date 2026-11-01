# Working on firede/gpui

Zed's GPUI at the commit a gpui-pre release snapshots (`vendor`), with a queue of patches on `main`.
Read `README.md` for the layout and `SYNC.md` before moving to a new gpui-pre release.

## Rules

- `vendor` holds only what `script/import-zed` makes from Zed. Never commit to it by hand. To change
  what the import takes, change `script/import-zed` or `fork.toml` and import again.
- `main` is `vendor`, one infrastructure commit (this repository's own files), and one commit per
  patch. Don't merge into it. Change a patch by amending its commit:
  `git commit --fixup <patch>` then `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash upstream/gpui-pre-<version>`.
- Release tags (`gpui-pre-<version>-p<n>`) are never moved or deleted: applications depend on them.
- The stand-ins in `compat/` are made by the import. A patch never changes them, and never changes
  public API that libraries built against gpui-pre rely on: it may only add to it.
- Write every patch as a Zed pull request would be written, so it can be proposed upstream as it is:
  - Change Zed's files as little as the fix needs. No reformatting, reordering or renaming of code the
    patch doesn't need to change.
  - Follow Zed's Rust guidelines (Zed's `CLAUDE.md`): no `unwrap()` where an error can be propagated,
    no `let _ =` on fallible calls, full words for names, comments only for why.
  - Subject `crate: Imperative summary`, as Zed's history has it. The body says what goes wrong
    without the patch and why the change fixes it.
  - End the message with two trailers: `Upstream:` (where the change is headed: a Zed pull request
    or issue, or what has to be discussed first) and `Drop-when:` (what in Zed makes the patch
    unnecessary).
  - Add a test when the behavior can be tested in GPUI's test platform.
- Keep the patch table in `README.md` matching the queue.
- `script/check` passes before `main` is tagged, on macOS and on Linux (`ssh parlor`).
