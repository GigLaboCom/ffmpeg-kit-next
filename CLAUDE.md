# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

`GigLaboCom/ffmpeg-kit-next` is a **fork of [arthenica/ffmpeg-kit-next](https://github.com/arthenica/ffmpeg-kit-next)**
that exists for **one purpose**: produce standalone **LGPL-3.0 `ffmpeg` / `ffprobe`
command-line binaries** for the **MnemoVi** desktop app (repo `GigLaboCom/mnemoria-lvkb`).

Upstream ffmpeg-kit is a *library* packager (it builds `libav*` frameworks to embed
in mobile/desktop apps via `--disable-programs`). It does **not** ship the ffmpeg CLI.
MnemoVi runs ffmpeg as a **signed subprocess sidecar** (never linked — linking would
drag copyleft into its Rust binary), so we patch the build to emit the CLI instead.

Work happens on branch **`mnemovi/lgpl-cli`**. Tag **`mnemovi-cli-v*`** triggers the
release build. See `MNEMOVI-LGPL-CLI.md` for the LGPL rationale.

## License posture (do not break)

- Build is **`--full` = all non-GPL libraries**, `--enable-version3` → **LGPL-3.0**.
- **Never** pass `--enable-gpl`. The GPL libs (`x264`, `x265`, `xvidcore`,
  `libvidstab`, `rubberband`) must never be built. kit-next hard-errors if a GPL lib
  is enabled without `--enable-gpl`; keep it that way.
- ffmpeg's TLS backend is **gnutls** (LGPL). openssl still builds (other `--full`
  libs depend on it) but ffmpeg passes `--disable-openssl` (see `scripts/apple/ffmpeg.sh`)
  because ffmpeg forbids both TLS backends at once.
- This repo is **public** so its Releases page is the LGPL source-availability offer.

## How the build works

`.github/workflows/mnemovi-lgpl-cli.yml` — free `macos-15` runners, an **arch matrix**
(`aarch64` + `x86_64`; x86_64 is cross-compiled, so its verify installs Rosetta):

1. Select **Xcode 26** (the `xcode26` Nix profile requires it; re-points
   `/Applications/Xcode.app`).
2. Install Nix, then `MNEMOVI_PROGRAMS="--enable-ffmpeg --enable-ffprobe --disable-ffplay"
   ./nix-macos.sh -p xcode26 --full --disable-{x86-64|arm64} --jobs=3 --no-output-redirection`.
3. `scripts/apple/ffmpeg.sh` builds ffmpeg **static** with programs, copies
   `ffmpeg`/`ffprobe` to `prebuilt/mnemovi-cli/<arch>/`, then **`exit 0`** — stopping the
   whole build before `ffmpeg-kit` / framework packaging (which we neither need nor build).
4. Verify (arch via `lipo`, LGPL via `-version`, self-contained via `otool -L` = system
   dylibs only) → publish **arch-suffixed** assets (`ffmpeg-<arch>`, `ffprobe-<arch>`,
   `PROVENANCE-<arch>.txt`) to the `mnemovi-cli-v*` Release.

Local build: `MNEMOVI_PROGRAMS="--enable-ffmpeg --enable-ffprobe --disable-ffplay"
./nix-macos.sh -p xcode26 --full --jobs=3` (needs Nix + Xcode 26).

MnemoVi consumes the Release assets via `~/self/mnemovi-lgpl-ffmpeg/02-swap-assets.sh`.

## Our changes vs upstream (keep this list current)

All gated so upstream behaviour is untouched unless `MNEMOVI_PROGRAMS` is set.

| File | Change | Why |
|---|---|---|
| `scripts/apple/ffmpeg.sh` | static build + programs, copy CLI out, `exit 0`; `--disable-openssl` | emit a self-contained LGPL CLI; resolve gnutls/openssl conflict |
| `flake.nix` | add `autogen ragel texinfo gtk-doc doxygen libtasn1` to `commonToolPackages`; alias `glibtoolize` | macOS Nix shell was leaner than Android's; `--full` libs (libsndfile/freetype/gnutls) need these tools |
| `scripts/apple/harfbuzz.sh` | `-Db_ndebug=true` | Xcode 26 libc++ removed `_LIBCPP_ENABLE_ASSERTIONS`, which harfbuzz's debug build defined |
| `.github/workflows/mnemovi-lgpl-cli.yml` | the CI (new file) | build + verify + release both arches |
| `MNEMOVI-LGPL-CLI.md` | docs (new file) | provenance + LGPL rationale |

Most of these are **Xcode-26 / newer-toolchain** fixes and **tool gaps** — candidates to
upstream as PRs (they help any macOS build), except the CLI-emit patch (MnemoVi-specific).

## Syncing with upstream (arthenica)

Upstream is added as remote `upstream`
(`https://github.com/arthenica/ffmpeg-kit-next`). To pull upstream changes onto our
patch branch:

```bash
git remote add upstream https://github.com/arthenica/ffmpeg-kit-next.git  # once
git fetch upstream
git checkout mnemovi/lgpl-cli
git rebase upstream/main        # prefer rebase — keeps our ~13 commits on top, replayable
# resolve conflicts (most likely in scripts/apple/ffmpeg.sh and flake.nix), then:
git push --force-with-lease origin mnemovi/lgpl-cli
```

Prefer **rebase over merge**: our changes are a small, ordered patch series meant to sit
on top of upstream, and the release build works off a clean tag. Because the patch is
also mirrored as a flat diff at `~/self/mnemovi-lgpl-ffmpeg/ffmpeg-kit-next-mnemovi-lgpl-cli.patch`,
a hard reset + `git apply` is a valid fallback if a rebase gets messy.

After syncing, cut a new release by moving/adding a tag:
`git tag -f mnemovi-cli-v<kitver> && git push -f origin mnemovi-cli-v<kitver>`.

## Raising the ffmpeg version

The ffmpeg source is pinned in **`scripts/source.sh`**, in the `ffmpeg)` case:

```sh
ffmpeg)
  SOURCE_REPO_URL="https://github.com/arthenica/FFmpeg"   # arthenica's ffmpeg mirror
  SOURCE_ID="n8.1.2"                                      # <-- the ffmpeg git tag
  SOURCE_TYPE="TAG"
  ;;
```

**That single `SOURCE_ID` is the ffmpeg version.** Upstream raises it exactly this way —
history shows `n7.1.5` at release `v7.1.0` → `n8.1.2` at `v8.1.0`; only the tag string
changes. To bump ffmpeg:

1. Confirm the target tag (e.g. `n8.2`) exists in `arthenica/FFmpeg` (they mirror the
   official FFmpeg release tags `nX.Y[.Z]`).
2. Edit `SOURCE_ID` in `scripts/source.sh`.
3. Re-run CI (push the `mnemovi-cli-v*` tag). Watch for new Xcode/SDK compile breaks —
   our fixes here are the pattern to follow.

Every other library is pinned the same way (its own `SOURCE_ID` in `source.sh`). Don't
switch a `SOURCE_REPO_URL` off arthenica's mirrors — the build scripts assume their layout
(e.g. gnutls rewrites `.gitmodules` to `arthenica/gnulib`).

## Conventions

- Don't remove the `${MNEMOVI_PROGRAMS:-...}` gating — it keeps upstream's default
  (library) build intact.
- Keep all edits **minimal and commented with a `MnemoVi:` prefix** so they're easy to
  spot during an upstream rebase.
- CI is **free only while this repo is public** — keep it public (also required for the
  LGPL offer).
- Never introduce an ffmpeg-sys / av* linkage anywhere downstream — ffmpeg is
  subprocess-only.
