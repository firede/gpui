# GPUI, as firede's apps use it

[Zed](https://github.com/zed-industries/zed)'s GPUI at the commit a [gpui-pre](https://crates.io/crates/gpui-pre)
release snapshots, with a short queue of patches the apps need before Zed has them. It is maintained
for NerdRaw and nerdice, not as a general-purpose fork.

Libraries (gpui-platinum, limu-gpui) depend on gpui-pre from crates.io, so they publish there and
build against Zed's code. An application that needs the patches replaces gpui-pre with this
repository through `[patch.crates-io]`: every crate in its build, libraries included, then uses one
GPUI, this one. For that, `compat/` holds a stand-in for each gpui-pre crate, with its name, version
and features, re-exporting the crate here it was published from.

Every patch is written the way a Zed pull request would be, and the queue is meant to shrink: a patch
is dropped once Zed has the change or something that replaces it.

## Layout

- `vendor`: one commit per gpui-pre release followed, made by `script/import-zed` and holding nothing
  but Zed's code at that release's commit (the GPUI crates and every crate they depend on through a
  path), the stand-ins, and a root `Cargo.toml`, `Cargo.lock` and toolchain made from Zed's. Tagged
  `upstream/gpui-pre-<version>`. Never edited by hand.
- `main`: `vendor`, then one commit with this repository's own files (this README, `SYNC.md`,
  `AGENTS.md`, `fork.toml`, `script/`), then one commit per patch. It is rebased onto each new
  import, so its history is rewritten.
- Release tags `gpui-pre-<version>-p<n>`: what applications depend on. A tag is never moved or
  deleted.

`ZED.toml` says which gpui-pre release and Zed commit the current base is. `script/patches` lists the
queue.

## Using it

A library depends on gpui-pre, at the version this repository follows:

```toml
[dependencies]
gpui = { package = "gpui-pre", version = "0.3.10" }
gpui_platform = { package = "gpui-pre-platform", version = "0.3.10", features = ["font-kit", "wayland", "x11"] }
```

An application depends on gpui-pre the same way, and replaces it with a release tag of this
repository:

```toml
[patch.crates-io]
gpui-pre = { git = "https://github.com/firede/gpui", tag = "gpui-pre-0.3.10-p1" }
gpui-pre-platform = { git = "https://github.com/firede/gpui", tag = "gpui-pre-0.3.10-p1" }
gpui-pre-macros = { git = "https://github.com/firede/gpui", tag = "gpui-pre-0.3.10-p1" }
```

List every gpui-pre crate its build has (`cargo tree -i <crate>` shows who brings one in); Cargo warns
about a listed one the build doesn't have. The patch applies only while the requirement in the
dependency graph matches the stand-in's version, which is gpui-pre's.

Cargo applies `[patch]` sections only from the root of a build, so an application also copies the
`[patch.crates-io]` section of this repository's `Cargo.toml`, at the tag it uses, into its own. Those
are Zed's fixes to crates.io crates GPUI uses (`async-task`, `calloop` and others); GPUI builds without
them but does not behave exactly as in Zed. The `[profile.dev.package]` overrides there (`taffy`,
`resvg` and the proc-macros at `opt-level = 3`) keep debug builds usable and are worth copying too.

On Windows, gpui's default `windows-manifest` feature embeds an application manifest with the
resource compiler. Cross-compiling for Windows from another OS needs `llvm-rc` for it, or the feature
turned off and the application providing its own manifest.

To build an application against a local checkout of this repository, patch the same crates with
paths in a `.cargo/config.toml` above the application (for example `~/Nerd/.cargo/config.toml`);
Cargo prefers a patch in configuration to one in `Cargo.toml`:

```toml
[patch.crates-io]
gpui-pre = { path = "/Users/firede/Nerd/gpui/compat/gpui-pre" }
gpui-pre-platform = { path = "/Users/firede/Nerd/gpui/compat/gpui-pre-platform" }
gpui-pre-macros = { path = "/Users/firede/Nerd/gpui/compat/gpui-pre-macros" }
```

## Patches

| Patch | Why | Headed for |
|---|---|---|
| `gpui: Add Window::paint_glyph_scaled` | Zooming text reuses its raster, scaled on the GPU with an unsnapped origin; without it, zooming drops about 30% of frames or text jumps in size | Zed, as a new public API after a discussion (related: zed-industries/zed#59413) |
| `gpui_linux: Offer clipboard metadata and images to other apps on Wayland` | Structured content across processes (a MIME type named by `GPUI_CLIPBOARD_METADATA_MIME`), and images on the clipboard | Zed: images as a pull request; custom formats need a new API, after a discussion |
| `gpui_windows: Compile embedded shaders at runtime` | Cross-compiling the Windows build from Linux or macOS | Zed |
| `gpui: Treat mouse up as mouse input` | After a key press during a drag, the release was dropped and the drag's last move lost | Zed (zed-industries/zed#64614) |
| `gpui: Make Window::with_element_opacity public` | Fading images that an element paints itself; `paint_image` takes no color | Zed, as a new public API after a discussion |

## Syncing with Zed

Started by hand, never on a schedule: a sync moves the libraries and every application to a new
GPUI, and their changes and tests go with it. [`SYNC.md`](SYNC.md) is the procedure.

## License

Apache-2.0, like the Zed crates this repository holds. Copyright Zed Industries, Inc.; the patches are
firede's.
