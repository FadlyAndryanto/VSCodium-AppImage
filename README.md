# VSCodium DWARFS AppImage Builder

This repository automatically packages the upstream **VSCodium stable AMD64 Debian package** as an **x86_64 AppImage**. GitHub Actions downloads the matching `.deb`, builds an AppDir, generates an `AppRun` launcher, and uses [`pkgforge-dev/appimagetool`](https://github.com/pkgforge-dev/appimagetool) to create a DWARFS-compressed AppImage with launch profiling.

Each build is published as a GitHub Release in this repository. The release tag is the upstream VSCodium version, for example `1.143.17232`.

## Features

- Checks for new VSCodium stable releases every three hours.
- Supports manual builds of the latest release or a specified version.
- Skips a version that already has a release, unless `rebuild` is enabled.
- Verifies the Debian package name and version before extracting it.
- Preserves the VSCodium-provided shell launcher at `usr/share/codium/bin/codium`.
- Generates `AppRun` inside the workflow; no separate launcher source file is needed in the repository.
- Downloads the latest stable `appimagetool` release and verifies its GitHub-provided SHA-256 digest when available.
- Enables DWARFS launch profiling and requires a profile to be produced before publishing.
- Publishes the AppImage, zsync metadata, `appinfo`, and SHA-256 checksums.

## Workflow file

The workflow is located at:

```text
.github/workflows/build-appimage.yml
```

It runs on `ubuntu-24.04` and targets `x86_64`/AMD64. The job has a 30-minute timeout and uses one job to avoid unnecessary runner startup and hand-off overhead.

### Automatic builds

GitHub Actions checks the latest stable VSCodium release every three hours, at minute 23:

```yaml
cron: "23 */3 * * *"
```

Scheduled workflow times use **UTC**, and GitHub may delay scheduled runs. The workflow reads the latest release tag from `VSCodium/vscodium`, removes an optional leading `v`, and accepts versions in `major.minor.patch` format. It then checks whether the same tag already has a release in this repository. Existing releases are skipped unless a rebuild was requested.

### Manual builds

1. Open the repository on GitHub.
2. Select **Actions**.
3. Open **Build VSCodium AppImage**.
4. Select **Run workflow**.
5. Choose the inputs and start the run.

| Input | Default | Purpose |
| --- | --- | --- |
| `version` | Empty | Optional VSCodium version or tag. Leave empty to build the latest stable release. A leading `v` is accepted. |
| `rebuild` | `false` | Set to `true` to rebuild a version even when its release already exists. |

For example, `1.143.17232` and `v1.143.17232` both select version `1.143.17232`. The workflow then checks that the corresponding AMD64 Debian package exists and matches the requested version.

When rebuilding an existing release, the workflow replaces assets with matching filenames using `gh release upload --clobber` and updates the release title and notes. Assets from older builds that have different filenames are not automatically deleted.

## Build process

The workflow performs these steps only when a build is needed:

1. **Resolve the release.** Select the manually requested version or query the latest upstream stable release. The existence of this repository's release is stored as a step output and reused during publishing, avoiding a second release lookup.
2. **Prepare launch profiling.** Check for `xvfb-run`, Xvfb, `xauth`, and FUSE helpers. Install only missing packages, so `apt-get update` is skipped when the runner already has the required tools. Verify `/dev/fuse`, make it accessible, and provide a `fusermount` compatibility link when needed. A private `XDG_RUNTIME_DIR` is also configured for the headless Electron launch.
3. **Install appimagetool.** Query the latest stable release from [`pkgforge-dev/appimagetool`](https://github.com/pkgforge-dev/appimagetool), select the exact `appimagetool-x86_64-linux` asset, and download its API-provided URL. When GitHub publishes an asset SHA-256 digest, the workflow verifies it before installation. The selected appimagetool version is recorded in the release notes.
4. **Download VSCodium.** Fetch `codium_<version>_amd64.deb`, verify that its package name is `codium` and its embedded version matches the selected release, then extract it into `AppDir/`.
5. **Prepare the AppDir.** Check the expected desktop file, icon, launcher, and application payload. Copy the desktop file and icon to the AppDir root, update the desktop entry to execute `AppRun`, and remove the package-specific `Path` entry.
6. **Generate `AppRun`.** Create the launcher directly inside `AppDir/`:

   ```sh
   #!/bin/sh
   set -eu

   if [ -z "${APPDIR:-}" ]; then
     APPDIR="$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)"
   fi
   export APPDIR
   exec "$APPDIR/usr/share/codium/bin/codium" "$@"
   ```

   This delegates startup to VSCodium's own shell wrapper, forwards all command-line arguments, and uses `exec` rather than adding another backgrounding layer.
7. **Build and profile the AppImage.** Run appimagetool with `--optimize-launch`, a 45-second profiling capture, the generated DWARFS profile path, and repository-specific zsync update information. The workflow fails if profiling does not produce a non-empty profile. It leaves `--dwarfs-comp` unset so appimagetool uses its built-in DWARFS compression defaults, including its AppDir-size-based block sizing.
8. **Validate outputs.** Require exactly one non-empty `.AppImage`, exactly one non-empty `.AppImage.zsync`, and a non-empty `appinfo` file. Then create `SHA256SUMS` for those three files.
9. **Publish the release.** Create a release for a new version, targeting the workflow's current commit, or upload/replace assets and update an existing release when rebuilding.

## Release assets

Each successful build is expected to publish:

| File | Description |
| --- | --- |
| `*.AppImage` | Executable x86_64 VSCodium AppImage. |
| `*.AppImage.zsync` | zsync metadata for compatible incremental-update tools. |
| `appinfo` | AppImage metadata generated by appimagetool. |
| `SHA256SUMS` | SHA-256 checksums for the AppImage, zsync file, and appinfo file. |

Published builds are available from the repository's **Releases** page.

## Permissions and efficiency choices

The workflow declares:

```yaml
permissions:
  contents: write
```

It uses the built-in `${{ github.token }}` through `GH_TOKEN` and sets `GH_REPO` to `${{ github.repository }}`. This lets GitHub CLI query and publish releases without checking out the repository, saving a checkout step while retaining explicit repository context. No personal access token is required.

The `vscodium-appimage-release` concurrency group prevents two runs from publishing the same repository's releases at the same time. `cancel-in-progress: false` avoids interrupting an active build or release upload.

The workflow resolves the latest stable appimagetool release on each build rather than pinning one version. This keeps the build on the current stable tool, but means a rebuild at a later date may use a newer appimagetool version; the exact version used is included in the release notes.

## Portability note

The workflow extracts the Debian package with `dpkg-deb -x`; it does **not** harvest or bundle every system runtime dependency. The resulting AppImage may still need compatible libraries from the host Linux system, for example GTK, NSS, or ALSA libraries.

The output is for **64-bit x86 (AMD64/x86_64)** only. This workflow does not build ARM images or guarantee operation on every Linux distribution without additional host libraries.

## Troubleshooting

- **The run skips the build:** A release with the selected version tag already exists. Run manually with `rebuild: true` to replace its assets.
- **The Debian package download fails:** Confirm that the upstream VSCodium release contains `codium_<version>_amd64.deb`.
- **A required package file is missing:** VSCodium may have changed its Debian package layout. Check the file paths verified in `Prepare AppDir and generate AppRun`.
- **Profiling fails because FUSE is unavailable:** The launch-profile pass requires `/dev/fuse`. The workflow stops rather than silently producing an unprofiled AppImage.
- **The appimagetool asset cannot be found:** Check the latest `pkgforge-dev/appimagetool` release for the exact `appimagetool-x86_64-linux` asset name.
- **Release publishing fails:** Check that repository Actions settings permit the workflow token to write repository contents.
- **The AppImage does not start on a Linux system:** Check for missing host libraries. This workflow does not bundle all runtime dependencies.

For build errors, open the failed run under **Actions** and inspect the first step that reports an error.
