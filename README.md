# UniCam Hub Releases

Public release artifacts for [UniCam Hub](https://unicam.app), the video
recording hub for IP cameras, RTSP streams, and video recording devices.

This repository contains no source code. It exists only to host downloadable
builds under [Releases](https://github.com/nittbit/unicam-releases/releases).

## Downloads

Each release ships the following assets:

| Platform | Asset |
|---|---|
| macOS (Apple Silicon) | `unicam-hub_<version>_arm64.dmg` |
| macOS (Intel) | `unicam-hub_<version>_amd64.dmg` |
| Windows x64 | `unicam-hub_<version>_amd64_setup.exe` (installer) or `unicam-hub_<version>_amd64.zip` (portable) |
| Linux amd64 | `unicam-hub_<version>_amd64.deb` |
| Linux arm64 | `unicam-hub_<version>_arm64.deb` |
| Homebrew tarballs | `unicam-hub_<version>_<os>-<arch>.tar.gz` |

Releases marked **Pre-release** are beta builds from the prerelease channel.
Releases without that mark are stable.

## Homebrew

```sh
brew install nittbit/tap/unicam-hub
brew services start unicam-hub
```

Formula source: [nittbit/homebrew-tap](https://github.com/nittbit/homebrew-tap).

## Verifying downloads

Every release includes a `manifest.json` listing the SHA-256 of each asset.
