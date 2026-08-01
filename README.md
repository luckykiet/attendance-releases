# Aviza terminal releases

Installers for the **Aviza attendance terminal**, the kiosk app that identifies
an employee at a fingerprint reader and records their attendance.

This repository holds **only the built installers**. The source is private; this
is public so that a kiosk can update itself without carrying a credential.

> Nothing here is usable on its own. A terminal is inert until it is paired with
> a register by somebody who can log in to that retail, so a downloaded installer
> grants nothing.

## Install a terminal

| Platform | File | Install |
|---|---|---|
| Windows (x64) | `Aviza-<version>.msi` | double-click, or `msiexec /i "Aviza-<version>.msi"` as administrator |
| Debian / Ubuntu | `attendance-terminal_<version>_amd64.deb` | `sudo apt install ./attendance-terminal_<version>_amd64.deb` |
| Fedora / RHEL | `attendance-terminal-<version>-1.x86_64.rpm` | `sudo dnf install ./attendance-terminal-<version>-1.x86_64.rpm` |

**The fingerprint reader needs an x64 (Intel/AMD) host.** ZKTeco ships its driver
as x86/x64 only — no ARM64 build, and Windows does not emulate kernel drivers, so
a terminal cannot read fingers on Windows-on-ARM (including a Windows VM on an
Apple Silicon Mac). Everything else runs anywhere.

Upgrading is installing the newer package over the old one. Pairing, queued
punches and local records survive it — they live outside the install directory,
in `%APPDATA%\AttendanceTerminal` / `~/.local/share/attendance-terminal`.

## How a kiosk updates itself

An installed terminal checks here every six hours, downloads the installer for
its platform, verifies it, and stages it for a privileged helper to install
unattended. That is what `latest.txt` below is for.

The client has no JSON parser — deliberately, it is a Swing app that speaks
tab-separated text to its own server too — so it reads one small manifest rather
than the GitHub API. `/releases/latest/download/<name>` always resolves to the
newest release, which means no API, no token, and nothing to configure on a
kiosk.

## `latest.txt`

Every release must carry this file as an asset. It is the whole protocol.

```
version	1.0.1
windows	Aviza-1.0.1.msi	9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
deb	attendance-terminal_1.0.1_amd64.deb	60303ae22b998861bce3b28f33eec1be758a213c86c93c076dbe9f558c11c752
rpm	attendance-terminal-1.0.1-1.x86_64.rpm	fcde2b2edba56bf408601fb721fe9b5c338d10ee429ea04fae5511b68fbf8fb9
```

Tab-separated. One `version` row, then one row per platform: **key, filename,
SHA-256**.

Rules a kiosk enforces, so a release has to respect them:

| Rule | What happens if you break it |
|---|---|
| `version` is `major.minor.patch`, nothing else | no kiosk ever updates to it |
| Every platform row carries a 64-character SHA-256 | that platform is skipped — nothing is downloaded that cannot be verified |
| The version matches what the installers were built with | the helper refuses to install the mismatch |
| Filenames match the assets actually attached | the download 404s and the check retries |

Unknown rows are ignored, so a future asset type is safe to add.

Ambiguity always means **do not update**. A kiosk that reinstalls itself on every
check is worse than one that never updates at all, so anything unreadable — a
version with a suffix, a missing digest, a truncated file — is treated as
"nothing to do" rather than guessed at.

## Cutting a release

1. Bump `version` in `terminals/core/app.properties` in the source repo. That one
   value becomes the installer's filename, the version in Add/Remove Programs and
   the version the running app reports.
2. Build the installers **on their own platforms** — jpackage only ever targets
   the OS it runs on:
   ```bat
   package-all.bat        REM Windows -> dist\Aviza-<version>.msi
   ```
   ```bash
   ./package-all.sh       # Linux -> dist/attendance-terminal_<version>_amd64.deb, dist/attendance-terminal-<version>-1.x86_64.rpm
   ```
3. Collect all three into **one** `terminals/dist/`, then let the source repo's
   release script write the manifest and publish:
   ```bash
   ./release.sh              # writes dist/latest.txt and shows it
   ./release.sh --publish    # ...and creates or updates the GitHub release
   ```

`release.sh` is the only supported way to produce `latest.txt`. It refuses to
publish an installer left over from an earlier build — the commonest way to cut a
broken release is a stale file in `dist/`, which yields a manifest naming the new
version beside an old package, and every kiosk then refuses it correctly and
invisibly for as long as nobody looks.

Re-running `--publish` on an existing tag uploads into it, so a release is
legitimately finished in two passes — one per build machine.

## Verifying a download by hand

```bash
sha256sum attendance-terminal_1.0.1_amd64.deb
```
```powershell
Get-FileHash Aviza-1.0.1.msi -Algorithm SHA256
```

Compare against `latest.txt` from the same release. A kiosk does this twice —
once when it downloads, and again in the privileged helper before it installs,
against the manifest the helper fetches itself.

## Reporting a problem

Issues belong in the private source repository, not here. This one carries
artifacts only.
