# Syncing with Zed

A sync moves `main` onto the Zed commit of a newer gpui-pre release and cuts a new release tag. It is
always started by hand, because the libraries and applications move with it; nothing here runs on a
schedule.

The examples move from gpui-pre `0.3.10` to `0.3.11`, with Zed cloned at `~/Store/Repo/zed`.

## 1. Choose the gpui-pre release

```sh
curl -s -A firede-gpui https://crates.io/api/v1/crates/gpui-pre | python3 -c \
  'import json,sys; [print(v["num"], v["created_at"][:10], v["description"]) for v in json.load(sys.stdin)["versions"][:5]]'
```

gpui-pre is published about once a week from Zed's `main`; its description names the Zed commit.
Before moving, check what pins gpui-pre exactly: gpui-component pins `=<version>`, so an application
using it can only move once a gpui-component release pins the new version.

## 2. Read what changed

```sh
old=$(sed -n 's/^commit = "\(.*\)"$/\1/p' ZED.toml)
git -C ~/Store/Repo/zed fetch origin
git -C ~/Store/Repo/zed log --oneline "$old..<new commit>" -- $(sed -n 's/^    "\(crates\/.*\|tooling\/.*\)",$/\1/p' ZED.toml)
```

Note renamed or removed public API (Zed renames things freely, e.g. `ScrollWheelEvent` to
`ScrollEvent`), changes to the files the patches touch, and any Zed commit that does what a patch
does.

## 3. Import it

```sh
script/import-zed --zed ~/Store/Repo/zed 0.3.11
```

This adds a commit to `vendor` and tags it `upstream/gpui-pre-0.3.11`. If it fails because a crate
needs something the import doesn't carry over (a new path dependency outside `crates/`, a file under
`assets/` reached in a new way, a patched crate with a local path, a new gpui-pre crate or feature),
fix `script/import-zed` or `fork.toml` on `main`, delete the commit and tag it made, and import
again.

## 4. Rebase the queue

```sh
git rebase --onto upstream/gpui-pre-0.3.11 upstream/gpui-pre-0.3.10 main
```

For each conflict, port the patch to Zed's new code, keeping its intent and its commit message. Then,
for every patch (`script/patches`):

- Check its `Drop-when` trailer against the new code, and the state of what its `Upstream` trailer
  names (`gh pr view`, `gh issue view`). Drop the patch when Zed has the change or something that
  replaces it, and say in the sync's summary what replaces it.
- If its commit message still describes it, leave the message alone.

## 5. Check

```sh
script/check                                  # on macOS: also the Windows build
ssh parlor 'cd ~/Nerd/gpui && git fetch && git checkout --detach origin/main && script/check'
```

`parlor` is the Linux machine. Push `main` first (step 6 without the tag) or copy the tree over.

## 6. Update the README and tag

Update the patch table and the versions in the examples in `README.md` (amend the infrastructure
commit, the first after `vendor`), then tag and push:

```sh
git tag gpui-pre-0.3.11-p1
git push --force-with-lease origin main vendor
git push origin upstream/gpui-pre-0.3.11 gpui-pre-0.3.11-p1
```

A change to the queue on the same base (a new patch, a fix to one) gets the next tag on that base,
`gpui-pre-0.3.11-p2`.

## 7. Update the libraries, then the applications

Each project moves as a change of its own:

- Libraries (limu-gpui, gpui-platinum): move to the new gpui-pre version, fix what Zed's API changes
  broke, test, release.
- Applications (NerdRaw, nerdice): move to the new gpui-pre version and the libraries' new releases,
  point `[patch.crates-io]` at the new tag, copy this repository's `[patch.crates-io]` section at the
  new tag over their own, build, and test on the platforms they ship on.
