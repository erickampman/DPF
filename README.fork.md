# The private DPF fork

DPF (DISTRHO Plugin Framework) with the local patches that StageHand, Umpire
and the plugins need, in one place. Upstream's README.md, LICENSE (ISC) and
LICENSING.md apply. Why the fork exists, and how consumers use it: Umpire's
`docs/decisions.md`, "DPF fork".

- **This repo:** `erickampman/DPF` (private), cloned at `~/dev/dev-common/DPF`
  on the Linux PC. Remote `upstream` is `DISTRHO/DPF`.
- **pugl** (`dgl/src/pugl-upstream`) is a submodule pointing at a second
  private fork, `erickampman/pugl` (`~/dev/dev-common/pugl`; remote
  `upstream` is `DISTRHO/pugl`), because PMU's macOS patches are in pugl.
  Push the pugl fork before a DPF commit that pins it.
- **Consumers** mount this repo as a submodule (StageHand and Umpire at
  `external/dpf`), clone with `--recurse-submodules`, and set
  `git config push.recurseSubmodules check`.

## Base

Made 2026-10-14 from StageHand's vendored copy (`external/dpf`, itself copied
from PolygraphModularUnified's `dgl-ui/dpf` at PMU `ed3761e`). That copy is,
apart from the patches below, exactly:

- DPF `4238e1c7` (upstream `main`, 2025-10-23, "Merge branch 'develop'");
- pugl `5e2621d` (DPF's pin, 2025-09-06).

Each patch is its own commit on top, so `git log upstream/main..main` lists
them.

## Local patches

Search the source for "LOCAL PATCH": each one is marked with where it came
from.

From PMU ("PMU LOCAL PATCH"):

- `dgl/src/WindowPrivateData.cpp`: re-read the scale factor on every
  configure event, and call `onScaleFactorChanged()` when it changes. All
  platforms.
- pugl `src/mac.m`, `src/mac_gl.m` (in the pugl fork): macOS cross-display
  scale fixes. `dispatchExpose` reads the view's own screen's scale; the GL
  view reshapes and redraws when its backing scale changes.

From StageHand ("STAGEHAND LOCAL PATCH"; StageHand's `docs/macos.md`,
milestone 6):

- `dgl/src/pugl.cpp`, `puglSetResizable()`: on macOS, also sets the draw
  view's autoresizing mask, so a window made resizable after it's realized
  resizes its drawing with it.
- `distrho/extra/FileBrowserDialogImpl.hpp`, `.cpp`: on macOS, choosing a
  folder (`FileBrowserOptions::directories`; with an extension, a package of
  that type too), an extension filter (`FileBrowserOptions::extension`,
  through UTType), the suggested name when saving, the sheet begun at once
  rather than from `dispatch_async`, and `fileBrowserClose()` ending a sheet
  still open.

The fork's own ("DPF FORK LOCAL PATCH"):

- `dgl/src/pugl.cpp`, `puglSetMatchingBackendForCurrentBuild()`: with
  `DGL_SWAP_INTERVAL` defined, sets pugl's `PUGL_SWAP_INTERVAL` hint to it
  when DGL makes a view (OpenGL only). `0` turns vsync off. Undefined, the
  driver's default applies, as upstream. Umpire defines it as `0` (its
  `docs/ui-spike.md`, finding 6: a swap blocked for about a second under
  XWayland). pugl applies the hint on X11 (`GLX_EXT_swap_control`) and
  Windows; its macOS backend ignores it. Define it on the DGL library target,
  since `pugl.cpp` is compiled there:
  ```cmake
  dpf__add_dgl_opengl(...)
  target_compile_definitions(dgl-opengl PUBLIC DGL_SWAP_INTERVAL=0)
  ```

Still to come: the plugins' patches, including the CLAP wrapper's UMP
handling (Umpire's `docs/plugins-roadmap.md`).

## Each consumer's DGL_NAMESPACE

Every consumer sets its own (`StageHandDGL`, `UmpireDGL`, `StruckSynthDGL`,
...). On macOS pugl names its Objective-C classes after it, and two binaries
in one process with the same name clash.

## Updating from upstream

```bash
git fetch upstream
git merge upstream/main                  # or rebase the patch commits
```

If upstream moves its pugl pin, merge the new pugl commit into the pugl fork
first (keeping PMU's patches), push it, then pin that commit here. Keep this
file's lists in step with the patches.

## Checking a build on Linux

```bash
cmake -S . -B build -G Ninja -DDPF_EXAMPLES=ON
ninja -C build
```

Builds the example plugins, each with a JACK standalone where it has one; the
ones with a UI use DGL.

Checked 2026-10-14 on the Linux PC (Ubuntu, GNOME on Wayland, XWayland, Intel
UHD 630 with Mesa): the examples build (one upstream warning, in
`utils/lv2-ttl-generator`), and a DGL window's GLX swap interval is 1 without
`DGL_SWAP_INTERVAL` and 0 with `DGL_SWAP_INTERVAL=0`.
