# VSCodium DWARFS AppImage Builder

Automated GitHub Actions workflow that packages the latest upstream **VSCodium**
release into a launch-profiled, DWARFS-compressed **x86_64 AppImage** and
publishes it as a GitHub Release.

[![Build VSCodium AppImage](https://github.com/OWNER/REPO/actions/workflows/build-vscodium-appimage.yml/badge.svg)](https://github.com/OWNER/REPO/actions/workflows/build-vscodium-appimage.yml)

> Replace `OWNER/REPO` and the workflow filename above with your actual values.

---

## What it produces

For each VSCodium version, a GitHub Release tagged with the bare version
(e.g. `1.143.17232`) containing:

| Asset | Description |
| --- | --- |
| `*.AppImage` | The final AppImage (DWARFS compression + launch-profile optimization). |
| `*.AppImage.zsync` | zsync metadata for delta updates. |
| `appinfo` | AppImage metadata emitted by `appimagetool`. |
| `SHA256SUMS` | SHA-256 checksums for the AppImage, zsync file, and appinfo. |

The release title is `VSCodium <version> — DWARFS AppImage`, and the notes
record the upstream version, the `pkgforge-dev/appimagetool` tag used, and the
fact that a DWARFS launch profile was generated.

---

## How it works

1. **Resolve version** — Uses the requested version (manual runs) or the latest
   upstream release from [`VSCodium/vscodium`](https://github.com/VSCodium/vscodium).
   Validates the version matches `X.Y.Z`.
2. **Skip or build** — If a release for that version already exists and
   `rebuild` is not set, the job exits cleanly without building.
3. **Prepare environment** — Installs `xvfb`, `xauth`, `fuse3`, `libfuse2`,
   ensures `/dev/fuse` is available, and sets up `XDG_RUNTIME_DIR` for the
   profiling step.
4. **Fetch `appimagetool`** — Downloads the latest
   [`pkgforge-dev/appimagetool`](https://github.com/pkgforge-dev/appimagetool)
   (`appimagetool-x86_64-linux`).
5. **Fetch VSCodium** — Downloads `codium_<version>_amd64.deb` from the upstream
   release and extracts it into `AppDir`.
6. **Prepare `AppDir`** — Rewrites the desktop entry (`Exec=AppRun`,
   `Icon=codium`), installs the icon, and generates a minimal `AppRun` script.
7. **Build** — Runs `appimagetool` with:
   - `--optimize-launch`
   - `--profile-timeout 45`
   - `--dwarfs-profile <file>`
   - `--update-info gh-releases-zsync|…|latest|*-x86_64.AppImage.zsync`

   The build is validated: exactly one non-empty AppImage, one non-empty
   `.zsync`, and a non-empty DWARFS profile must exist.
8. **Publish** — Creates or updates the GitHub Release with all assets,
   overwriting existing files when `rebuild` is enabled.

---

## Triggers

### Scheduled

Runs every 3 hours (`23 */3 * * *`). Each run checks for a new upstream
VSCodium release and builds only if one hasn't been published yet.

### Manual

Dispatch from **Actions → Build VSCodium AppImage → Run workflow**.

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `version` | string | no | _(latest)_ | VSCodium version/tag to build, e.g. `1.143.17232`. A leading `v` is stripped. |
| `rebuild` | boolean | no | `false` | Rebuild and overwrite assets even if a release for this version already exists. |

---

## Auto-updates (zsync)

The embedded update information string is:

```text
gh-releases-zsync|OWNER|REPO|latest|*-x86_64.AppImage.zsync
```

AppImageUpdate-compatible clients can update existing installs in place using
the published `.zsync` file. Because the channel is `latest`, clients track the
newest release.

---

## Notes & assumptions

- **Runner**: `ubuntu-24.04`, x86_64 only.
- **Timeout**: 30 minutes per run.
- **Concurrency**: One build at a time; queued runs wait (`cancel-in-progress:
  false`).
- **Permissions**: `contents: write` is required to create/update releases.
- **Token**: Uses the automatic `GITHUB_TOKEN` (`GH_TOKEN`) for `gh` and
  `gh api` calls — no extra secrets needed.
- **Toolchain drift**: `appimagetool` is resolved to `latest` on each run.
  Because its release asset filename is stable
  (`appimagetool-x86_64-linux`), this is safe; a missing asset fails the job
  loudly.
- **FUSE**: The profiling step requires `/dev/fuse`; the workflow aborts early
  if it is unavailable.
- **Idempotency**: Re-running a scheduled job for an already-published version
  is a no-op unless `rebuild` is set.

---

## Local reproduction (outline)

The workflow mirrors a standard manual build:

```sh
# 1. Extract the upstream package
dpkg-deb -x codium_<version>_amd64.deb AppDir

# 2. Rewrite AppDir (desktop, icon, AppRun) per the workflow
# 3. Build
APPIMAGE_ARCH=x86_64 ARCH=x86_64 VERSION=<version> \
  appimagetool --verbose \
    --optimize-launch \
    --profile-timeout 45 \
    --dwarfs-profile out.dwarfsprofile \
    --update-info 'gh-releases-zsync|OWNER|REPO|latest|*-x86_64.AppImage.zsync' \
    AppDir --output dist
```

A display server (`xvfb-run` when headless) and FUSE are required for the
profiling step.

---

## Credits

- Upstream application: [VSCodium](https://github.com/VSCodium/vscodium)
- Packaging tool: [pkgforge-dev/appimagetool](https://github.com/pkgforge-dev/appimagetool)
