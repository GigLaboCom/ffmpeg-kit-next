# MnemoVi LGPL ffmpeg/ffprobe CLI — fork notes

This is a fork of [arthenica/ffmpeg-kit-next](https://github.com/arthenica/ffmpeg-kit-next)
that additionally emits the **standalone `ffmpeg` / `ffprobe` command-line
executables** MnemoVi runs as signed subprocess sidecars. Upstream builds
**libraries only** (`--disable-programs`), which MnemoVi cannot use.

## License: LGPL-3.0 (not GPL)

- Built with `--full` = **all non-GPL** libraries.
- The GPL libraries (`x264`, `x265`, `xvidcore`, `libvidstab`, `rubberband`) are
  **never** built: no `--enable-gpl` is passed, and kit-next's own validator
  hard-errors if a GPL lib is enabled without it.
- `--enable-version3` is set → the binaries are **LGPL version 3**.
- `ffmpeg` stays a **subprocess sidecar**, never linked into MnemoVi's Rust
  binary, so no copyleft reaches the app. Users may replace it via
  `MNEMORIA_FFMPEG` / `MNEMORIA_FFPROBE`.

## What this fork changes vs upstream

1. `scripts/apple/ffmpeg.sh` — when `MNEMOVI_PROGRAMS` is set:
   - `--disable-programs` becomes `${MNEMOVI_PROGRAMS}` (enables ffmpeg/ffprobe),
   - ffmpeg is built **static** (`--enable-static --disable-shared`) instead of
     shared frameworks, so the CLI is **self-contained** (kit-next's external
     libs are already static),
   - the framework-packaging path is skipped and the binaries are staged to
     `prebuilt/mnemovi-cli/aarch64/`.
   Upstream behaviour is unchanged when the var is unset.
2. `.github/workflows/mnemovi-lgpl-cli.yml` — free `macos-14` (arm64) CI that
   builds, verifies LGPL + self-containment, and publishes the binaries as
   Release assets.

## Build locally

```bash
MNEMOVI_PROGRAMS="--enable-ffmpeg --enable-ffprobe --disable-ffplay" \
  ./nix-macos.sh -p xcode26 --full --jobs=3
# -> prebuilt/mnemovi-cli/aarch64/{ffmpeg,ffprobe}
```

Then verify: `ffmpeg -version` must show `--enable-version3`, no `--enable-gpl`,
no `libx264/libx265/libxvid`; `otool -L` must list only `/usr/lib` + `/System`.

## Consumed by

MnemoVi's `~/self/ci-assets/mnemovi/bin/` → its `macos-build.sh` stages these
into `resources/bin`, signs each Mach-O, and notarizes the bundle.
