# Releasing PdfCraft

Every push to the `release` branch runs `.github/workflows/release.yml`. It builds **Windows**
artifacts only — the setup `.exe` (a WiX Burn bootstrapper around the MSI), the MSI itself and the
portable zip for x64 and arm64 — signs them with the certificates in the repository's `release`
environment, and creates or updates a **draft** GitHub Release named `PdfCraft v<version>`.
Nobody sees a draft until a maintainer publishes it.

The other platforms (macOS, Linux, FreeBSD, the web) keep being built and checked by CI, but they
are not part of the release-to-release pipeline; cut a Windows-only release from this workflow.

The pipeline was ported from PhotoCraft's. User-facing names say **PdfCraft**; files, binaries and
ids stay lowercase (`pdfcraft-<version>-<platform>-<arch>.<ext>`, `ai.storyteller.pdfcraft`).

## Cutting a release

1. **Bump the version** on `main`. It lives only in `[workspace.package] version` in the root
   `Cargo.toml`:

   ```sh
   cargo xtask version                 # prints the current version
   cargo xtask version set 0.3.0       # or 0.3.0-rc.1; updates Cargo.toml and Cargo.lock
   ```

   Commit the change through the normal review flow (see the `Release: PdfCraft v0.2.1` PR).
2. **Merge `main` into `release`** (or fast-forward it) and push. The workflow starts by itself.
3. **Wait for the draft.** When every job has finished (the ARM64 build is the slow part), the
   Releases page has a draft `PdfCraft v0.3.0` targeting the pushed commit, with every artifact
   and `SHA256SUMS.txt`. The notes are generated from the merged PRs.
4. **Check it.** Download an installer or two and read the job summaries. A `::warning::` there
   means a signing secret was missing and that artifact is unsigned.
5. **Publish** the draft in the GitHub UI. Publishing creates the `v0.3.0` tag. Versions with a
   pre-release suffix (`-rc.1`) are marked as pre-releases.

Pushing to `release` again before you publish rebuilds the same draft and replaces its assets.
Once the draft is published, the workflow refuses to touch that version again, so bump it first.

**Test runs:** *Actions ▸ Release ▸ Run workflow* runs the whole pipeline by hand. The optional
`version` input (such as `0.3.0-rc.1`) overrides `Cargo.toml` for that run only; the build applies
it with `cargo xtask version set` before building, so the binaries report it too. The Windows job
and the draft-release job use the `release` environment, so pick the `release` branch in the dialog
for a full run. On any other branch the same dispatch is a **dry run**: the `release` environment is
refused by its branch rule, nothing builds, and the draft-release job is skipped. The nightly
dry run on `main` (02:20 UTC) is the exception that still builds both architectures, unsigned, so
release-only breakage surfaces before release day (`gh workflow run release.yml --ref <branch>`).

## What gets built

| Platform | Artifacts | Built on |
|---|---|---|
| Windows 10+ x64 | `pdfcraft-<v>-windows-x64-setup.exe`, `pdfcraft-<v>-windows-x64.msi`, `pdfcraft-<v>-windows-x64-portable.zip` | `windows-latest` |
| Windows 11 on ARM64 | `pdfcraft-<v>-windows-arm64-setup.exe`, `pdfcraft-<v>-windows-arm64.msi`, `pdfcraft-<v>-windows-arm64-portable.zip` | `windows-11-arm` |

The setup `.exe` is the primary Windows artifact; the MSI and the portable zip remain too.
Both setups embed the signed MSI through a WiX Burn bootstrapper, so they install the same
payload, with the same install-policy, scope and shortcut contract.

The ARM64 Windows build runs natively on the `windows-11-arm` runner, so signing and WiX work
the same as for the x64 build; `.github/workflows/windows-arm64.yml` still installs and runs it
on ARM64 hardware on every change.

Every binary reports its version (`pdfcraft --version`, `pdfcraft-cli --version`, *Help ▸ About*).
The workflow sets `PDFCRAFT_BUILD_SHA` and `PDFCRAFT_BUILD_DATE` (`packaging/env.sh` fills them in
for local builds): the commit is recorded in the macOS `Info.plist` (`PdfCraftBuildCommit`) and the
date in the AppStream metadata. The binaries don't embed the commit yet.

**Fonts:** every job checks out [craft-fonts](https://github.com/storytold/craft-fonts) at the commit
pinned in `release.yml` and builds with `CRAFT_FONTS_DIR` and `CRAFT_FONTS_REQUIRED=1`, so releases
embed its Japanese fonts and fail rather than ship without them (docs/development.md "Fonts and attribution"). Bump the pin
deliberately.

**OCR models:** desktop packaging stages the Scan & OCR models (#103).
The packaging scripts call `stage_models` (`packaging/env.sh`; `package.ps1` on Windows), which runs `cargo xtask models` to fetch every `kind = "model"` file in `ATTRIBUTION.toml` and verify its SHA-256.
The current detection and recognition files are `text-detection.onnx` and `text-recognition.onnx`.
Each model ships with its licence text and the directory's `ATTRIBUTION.txt`, so packaging needs the network.
Upstream announced the legacy RTEN download URLs' brownout in [ocrs #282](https://github.com/robertknight/ocrs/issues/282) on 2026-10-27 and directs consumers to the ONNX models.

Runtime discovery prefers a complete ONNX detection/recognition pair within each directory, then a complete legacy `text-detection.rten` / `text-recognition.rten` pair.
An incomplete ONNX update can therefore use an intact RTEN pair in the same directory.
Discovery never combines model formats or files from different directories.
Directory precedence stays the same: `$PDFCRAFT_MODELS`, the release-package locations beside the executable, then the source tree's `assets/models/`.
The first directory with a complete supported pair wins, even if an earlier legacy pair precedes a later ONNX pair.

The release-package locations are `models\` beside `pdfcraft.exe` (MSI, portable zip), `PdfCraft.app/Contents/Resources/models` (macOS), and `share/pdfcraft/models` beside `bin/` (deb, rpm, tar.gz, AppImage, Flatpak, FreeBSD).
The web build does not include these models, and OCR remains limited to the Latin alphabet without accents.

### macOS

`packaging/macos/package.sh` builds both architectures, joins them with `lipo`, and assembles
`PdfCraft.app` from `Info.plist.in` (bundle id `ai.storyteller.pdfcraft`, macOS 11+, PDF declared
as a document type with rank Alternate, so PdfCraft is offered under Open With without taking over
from Preview). Files opened from Finder arrive as Apple events, which
`apps/pdfcraft/src/apple_events.rs` receives.

- **Signing** uses the hardened runtime and a secure timestamp, executable first, then the bundle.
  `packaging/macos/import-cert.sh` puts the certificate in a temporary keychain, deleted at the end
  of the job.
- **Notarization:** the app is zipped and sent with `xcrun notarytool submit --wait`, the ticket is
  stapled, and the result is checked with `codesign --verify`, `stapler validate` and `spctl`. The
  app ships on a drag-to-Applications DMG, which is signed and notarized too.
  Its Finder window (background, icon size and positions) comes from
  [`packaging/macos/dmg/`](../packaging/macos/dmg/README.md), and its volume is named `PdfCraft`
  without the version, which the window's background needs; the DMG file name keeps the version.
- **CLI:** `pdfcraft-cli` is signed and notarized as a zip. A bare executable can't hold a stapled
  ticket, so Gatekeeper looks it up online the first time a downloaded copy runs.

Without certificates (locally) the script signs ad-hoc and skips notarization:

```sh
packaging/macos/package.sh --arch aarch64     # quicker, host-only; --arch universal needs both targets
```

### Windows

`packaging/windows/package.ps1 -Arch x64|x86|arm64` builds with `-C target-feature=+crt-static`, so
neither the MSI nor the portable zip needs the Visual C++ redistributable.

- Before packaging, it reads both executables' PE headers: the machine type must match `-Arch`, and
  `pdfcraft.exe` must be a GUI-subsystem program (no console window, #57) while `pdfcraft-cli.exe`
  stays a console program.
- `pdfcraft.wxs` (WiX v5) is a dual-scope package (per-user by default into
  `%LOCALAPPDATA%\Programs\ArtCraft\PdfCraft`, per-machine into `Program Files\ArtCraft\PdfCraft` (`Program Files (x86)` for the 32-bit build),
  see `packaging/windows/INSTALLATION.md`) with a Start Menu shortcut, a
  desktop shortcut (on by default) and an App Paths entry. Both shortcuts are plain links to
  `pdfcraft.exe`, not advertised MSI shortcuts (#107, #143). The MSI version is the numeric `X.Y.Z`
  (MSI has no pre-release field), and same-version upgrades are allowed so release candidates replace
  each other. Icon ids end in `.ico` or `.exe` (Windows Installer requires it); packaging-lint checks
  this.
- `installer-ui.wxs` supplies native welcome, maintenance, progress, files-in-use and outcome
  dialogs. Full UI confirms success with Finish; failures and cancellations have distinct messages.
  The welcome dialog has an "Install to" row (a `PathEdit`, and Browse… into `BrowseFolder`, a
  native folder browser: drive list, folder list, Up, New folder, typed path; #1139), a "Create a
  desktop shortcut" checkbox and an "Add Edit and Combine files to the right-click menu of PDF
  files" checkbox, both ticked by default. Install sets the folder (`SetTargetPath`) before the
  dialog closes. `INSTALLFOLDER` is secure, so the choice reaches the elevated install, and
  `INSTALLFOLDER=<path>` works on the command line. The row starts on the folder
  `install-policy.wxi` chose for the scope picked on the first page (the recorded folder on an
  update), and an update installs into the recorded folder unless the command line gives one
  (`INSTALLATION.md`). `/qn` and `/qb` stay unattended and create both unless
  `INSTALLDESKTOPSHORTCUT=0` or `INSTALLPDFCONTEXTMENU=0` is passed;
  `INSTALLIMAGECONTEXTMENU=0` (no checkbox) leaves out the image menu. An update (the launcher runs
  `msiexec /passive` with no properties) keeps the desktop shortcut only if `PdfCraft.lnk` is still
  on the all-users desktop, so deleting it once keeps it away (#1058), and keeps each menu only if
  its key path is still in HKLM (#305). Both are AppSearch results read before the old version is
  removed. An update from a release that didn't have a menu yet (before 0.7.1 for the PDF menu,
  before 0.4.0 for the image menu, found by `OnlyDetect` Upgrade rows) adds it like a first install.
  A value given on the command line wins.
- The PDF right-click entry (#682) is a classic verb under
  `HKLM\Software\Classes\SystemFileAssociations\.pdf\shell\PdfCraft.Edit`, so it shows whichever app
  is the default for `.pdf` and never changes that default. Several selected PDFs (up to 15, the
  limit for a `Document` verb) start one `pdfcraft.exe "<file>"` each, and the single-instance
  hand-off opens them as tabs. On Windows 11 it is under "Show more options"; a first-level entry
  needs an `IExplorerCommand` handler and package identity (a sparse MSIX package), which the MSI
  doesn't have. "Combine files in PdfCraft" (#585) is a second verb in the same component,
  `PdfCraft.Combine`, running `pdfcraft --combine "<file>"` once per selected file; the
  single-instance hand-off sends them with `"combine": true`, and the running app adds them to
  its Combine files tab (`OsEvent::Combine`). Both verbs come and go with `INSTALLPDFCONTEXTMENU`.
- The image right-click entry (#190) is the same kind of verb, `PdfCraft.CreatePdf`, for ten image
  types, in its own component (`PdfcraftImageContextMenu`) since it became optional. Its rows moved
  out of `PdfcraftApp`; that's safe because `MajorUpgrade` removes the old version before installing
  the new one (`afterInstallValidate`), which `test-msi.ps1` checks.
- `test-msi.ps1 <file.msi>` checks the compiled shortcut, context-menu and dialog tables, and
  evaluates the first-install and update conditions of the desktop shortcut and both menus. Packaging
  runs it in a child process before signing, so the MSI isn't held open when signtool runs. The ARM64 install
  smoke test checks that both all-users shortcuts point at the installed `pdfcraft.exe` and are
  removed on uninstall, and that `INSTALLDESKTOPSHORTCUT=0` skips the desktop one.
- The portable zip holds both executables, the README, the licences, the OFL licence of each
  embedded craft-fonts family, and `portable.txt`. That marker beside `pdfcraft.exe` keeps the
  settings, logs, crash recovery and new digital IDs in `PdfCraftData\` next to the exe instead of
  `%APPDATA%` and `%LOCALAPPDATA%` (`crates/ui-egui/src/platform/portable.rs`). The MSI carries the same
  OFL notices; `-SkipBuild` requires `CRAFT_FONTS_DIR` so they can be staged, or
  `-BuiltWithoutCraftFonts` for binaries built without that input, and `test-msi.ps1` checks that
  the compiled MSI has OFL notice rows when expected (and none for a no-font build).
  Before either package is created, `font-notices.ps1` validates every declared OFL notice in
  `fonts/manifest.txt` and the native `fonts/simplified-chinese-manifest.txt` when present
  (required when `CRAFT_FONTS_REQUIRED` is set). Missing or empty family notices fail packaging;
  one family's notice cannot cover another. The check conservatively includes declared faces
  omitted by build-time size caps. `test-font-notices.ps1` exercises these failures in CI.
  The supplied checkout must match the existing binaries when using `-SkipBuild`.
  That presence check does not establish byte-for-byte payload or installation coverage;
  final package qualification still checks all notices against the staged files.
- **Setup `.exe`:** `packaging/windows/setup.wxs` is a WiX Burn bootstrapper around the signed MSI
  (`pdfcraft-<version>-windows-<arch>-setup.exe`). It carries the internal UI bootstrapper
  application from the WixToolset.BootstrapperApplications.wixext extension (WiX has no default BA),
  so the file is a single self-contained executable a user can double-click and it drives the MSI's
  own dialogs — same install policy, scope, shortcut and right-click-menu contract as the MSI; the
  MSI is embedded and hidden inside the bundle (`Visible="no"`), so *Programs and Features* lists one
  PdfCraft entry — the bundle — not the MSI. The bundle's version is the MSI version padded to four
  parts, and its `UpgradeCode` is distinct from the MSI's, so upgrading the setup `.exe` upgrades the
  installed bundle and lets the MSI's normal MajorUpgrade replace the old MSI in place. Building it
  needs `wix extension add -g "WixToolset.BootstrapperApplications.wixext/5.0.2"` once per machine;
  `package.ps1 -SetupExe` gates the step on that, and the release workflow installs it before
  packaging.
- **Signing:** `packaging/windows/sign.ps1` signs both executables, the MSI and the setup `.exe`
  with `signtool` (SHA-256, RFC 3161 timestamp), from a `.pfx` (`WINDOWS_CERTIFICATE`) or Azure
  Trusted Signing (`AZURE_*`), whichever is configured. It is the one place to change when signing
  changes.

Locally: `dotnet tool install -g wix --version 5.0.2` (and, for the setup `.exe`, `wix extension add -g "WixToolset.BootstrapperApplications.wixext/5.0.2"`), then `pwsh packaging/windows/package.ps1 -Arch x64` (add `-SetupExe` for the `.exe` setup file).

### Linux

`packaging/linux/package.sh` stages one FHS tree (both binaries, the desktop entry, hicolor icons,
AppStream metainfo) and makes every format from it: an **AppImage** (any distribution, nothing to
install), a **.deb** and an **.rpm** (built with [nfpm](https://nfpm.goreleaser.com) from
`nfpm.yaml`; they integrate with the menu, MIME and icon caches), a **.tar.gz**, and a CLI-only
**`pdfcraft-cli-<v>-linux-<arch>.tar.gz`** (the stripped `pdfcraft-cli` plus the licences and
README, for servers, CI and agent sandboxes). `--formats` picks a subset (`appimage deb rpm tar cli`).

The binaries are built on Ubuntu 22.04, the oldest GitHub-hosted image, so they need only
**glibc ≥ 2.35**: Ubuntu 22.04+, Debian 12+, Fedora 36+, RHEL 10. Windowing (X11, Wayland,
xkbcommon) and the GPU (Vulkan, EGL) are loaded at runtime from the system; the .deb and .rpm declare
them as dependencies (see `nfpm.yaml`).

Each AppImage embeds update information
(`gh-releases-zsync|storytold|pdfcraft|latest|pdfcraft-*-linux-<arch>.AppImage.zsync`), and its
`.zsync` is published beside it, so AppImageUpdate and AppImageLauncher can update it in place,
downloading only the changed blocks. `latest` is the newest published, non-pre-release version.
`package.sh` writes the `.zsync` when `zsyncmake` (the `zsync` package) is installed and warns
otherwise; the release job checks both.

**Flatpak:** the `flatpak` job turns each arch's tarball into a single-file bundle with
`packaging/linux/flatpak-bundle.sh` and `flatpak/ai.storyteller.pdfcraft.bundle.yml` (no Rust build;
the same binaries), then installs it and runs `pdfcraft-cli --version` in the sandbox.
`flatpak/ai.storyteller.pdfcraft.yml` is the from-source manifest for a later Flathub submission;
packaging-lint keeps the runtime and sandbox permissions (`finish-args`) of the two identical.
Printing is not available in the Flatpak yet: it runs `lp`, which the freedesktop runtime lacks
(the job logs a note); it needs the print portal. Users install the bundle with `flatpak install --user pdfcraft-<v>-linux-<arch>.flatpak`; the
freedesktop runtime comes from Flathub.

Locally (on Linux, with nfpm): `packaging/linux/package.sh` or `--formats "deb tar"`; then
`packaging/linux/flatpak-bundle.sh` for the Flatpak.

### FreeBSD

GitHub has no FreeBSD runners, so the `freebsd` job runs `packaging/freebsd/package.sh` in a FreeBSD
VM with the packages from `.github/workflows/freebsd.yml`. The tarball is a `/usr/local`-style tree
with the same desktop entry, metainfo and icons as Linux. FreeBSD has no code signing for loose
binaries; check the tarball against `SHA256SUMS.txt`.

### Web

`packaging/web/package.sh` runs `trunk build --release` in `apps/pdfcraft-web` and zips the site
with sample `_headers` and `.htaccess` files. Hosting (MIME types, compression, caching, iframes) is
covered in [`packaging/web/README.md`](../packaging/web/README.md).

## Secrets

The signing secrets live in the repository's **`release` environment** (*Settings ▸ Environments ▸
release*), restricted to the `release` branch; the Windows job and the draft-release job declare
`environment: release`. The release is unsigned (and the `release` environment refuses the run)
when neither credential source is set: the build refuses to ship an unsigned Windows release.

| Secret | Used for |
|---|---|
| `WINDOWS_CERTIFICATE`, `WINDOWS_CERTIFICATE_PASSWORD` | base64 `.pfx` code-signing certificate and its password |
| `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET` | service principal for Azure Trusted Signing (instead of a `.pfx`) |
| `AZURE_SIGNING_ENDPOINT`, `AZURE_SIGNING_ACCOUNT`, `AZURE_CERT_PROFILE` | Trusted Signing endpoint, account and certificate profile |

`GITHUB_TOKEN` creates the release; only the final job gets `contents: write`.

## Checks

`.github/workflows/packaging-lint.yml` runs on changes to `packaging/` and the workflows: actionlint, shellcheck,
a PowerShell parse, xmllint (WiX, plist, MIME), `desktop-file-validate`, `appstreamcli validate`, and
the MSI icon-id check. The FreeBSD and Windows ARM64 workflows also exercise their packaging before a
release does.

## Scanner detections

Antivirus and reputation scanners sometimes flag a released file. Check the facts before you change
code or answer publicly. The first report was [issue #747](https://github.com/storytold/pdfcraft/issues/747),
a match of the `CobaltStrikeBeacon` family rule.

1. **Confirm the file is the release.** Compare its SHA-256 with `SHA256SUMS.txt` on the GitHub
   Release. Only a file whose hash matches the release says anything about that release.
2. **Record the signature.** `packaging/windows/security-audit.ps1` prints the Authenticode status:
   - `Valid`: the signature verifies.
   - `NotSigned`: no signature, as in developer and test builds. Not evidence of compromise by itself.
   - `HashMismatch`: the file changed after it was signed. Investigate.
   - `NotTrusted` or `UnknownError`: a signature exists but did not validate on this machine. Compare
     the signer and the hash with the release.
3. **Extract the payloads without running them.** Scan the executables inside the MSI and the
   portable zip, not only the outer installer. `7z x -oDIR pdfcraft-<version>-windows-x64.msi`
   extracts them. The payload names are WiX-internal, so tell the executables apart by their file
   type. Don't install or run the MSI or the extracted files.
4. **Scan with YARA and keep every match.** Use your own copies of YARA and the rule. PdfCraft
   bundles neither, and the helper never runs in the app, the installer or CI:

   ```powershell
   pwsh packaging/windows/security-audit.ps1 -Path .\pdfcraft.exe, .\pdfcraft-cli.exe -YaraExe C:\tools\yara64.exe -YaraRule C:\rules\CobaltStrikeBeacon.yar
   ```

   The output lists each matched identifier with its offset, and the SHA-256 of the rule file.
   Record where the rule came from and its revision. For CAPEv2's `CobaltStrikeBeacon` rule, that is
   the commit that last changed `data/yara/CAPE/CobaltStrikeBeacon.yar`.
5. **Find the condition branch.** Family rules usually have alternative branches, such as
   `all of ($x*) or 2 of ($a*) or ...`. Copy the rule, make one rule per branch with only the strings
   it uses, and scan again to see which branch fired.
6. **Interpret with care.** A match on generic repeated-byte patterns says much less than a match on
   distinctive code or strings, and it does not show that the software belongs to the family. Rules
   change too: CAPEv2 commit `9b599185` (2024-12-05) lengthened this rule's two repeated-byte patterns
   from 4 to 16 bytes to reduce false positives, so an older rule revision can match files that the
   current one does not. Don't alter PdfCraft's binaries to change a scanner's result. That changes
   nothing about the software.
7. **Report.** To the scanner, send the file's SHA-256, the release URL, the rule name and revision,
   and the matched identifiers and offsets. Report PdfCraft problems at
   <https://github.com/storytold/pdfcraft/issues> with the same facts.
