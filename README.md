# webrtc-build

Build scripts and CI for [webrtc-sdk/webrtc](https://github.com/webrtc-sdk/webrtc) binaries.

## Outputs

| Target | Host | Output |
|---|---|---|
| `apple` | macOS | `WebRTC.xcframework.zip` |
| `apple_prefixed` | macOS | `LiveKitWebRTC.xcframework.zip` |
| `apple_prefixed_stripped` | macOS | `LiveKitWebRTC-stripped.xcframework.zip` |
| `android` | Linux | `libwebrtc.aar` |
| `android_prefixed` | Linux | `libwebrtc_prefixed.aar` |
| `android_prefixed_stripped` | Linux | `libwebrtc_prefixed_stripped.aar` |
| `libwebrtc_linux_{x64,arm64}` | Linux | `libwebrtc-linux-<arch>-<config>.zip` |
| `libwebrtc_win_{x64,arm64}` | Windows | `libwebrtc-win-<arch>-<config>.zip` |
| `rust_<triple>` | Linux, macOS, Windows | `webrtc-<triple>-<config>.zip` (see below) |

Every output is written to `_package/` with a `<name>.shasum` file next to it (`shasum -a 256` format).

## Building

```sh
# All targets passed together share one checkout and one gclient sync.
python3 scripts/build.py build apple apple_prefixed apple_prefixed_stripped
python3 scripts/build.py build android android_prefixed android_prefixed_stripped

# libwebrtc wrapper builds need a webrtc-sdk/libwebrtc checkout.
python3 scripts/build.py build libwebrtc_linux_x64 --libwebrtc-dir ../libwebrtc

# livekit/rust-sdks webrtc-sys prebuilts need a livekit/rust-sdks checkout.
python3 scripts/build.py build rust_linux_x64 rust_android_arm64 --rust-sdks-dir ../rust-sdks

# Build and run rtc_unittests (debug) for the host platform.
python3 scripts/build.py test
```

Options:

- `--commit <sha|tag|HEAD>`: the webrtc commit to build. Defaults to `WEBRTC_COMMIT` in [VERSION](VERSION).
- `--debug`: debug build.
- `--no-sync`: rebuild the existing checkout without fetching (local iteration).
- `--work-dir <dir>`: where `_source/` and `_package/` go (the default is the repository root).
- `--jobs <n>`: ninja parallelism.
- `--keep-out`: keep build outputs and the patched tree for incremental local builds. By default every build output under `src/out` is deleted once copied to `_package`, and the tree is reset on exit, so cached checkouts only hold sources.
- `--fresh`: delete the checkout and sync from scratch.
- `--sync-for <targets>`: also sync the dependencies of these targets (keeps shared caches identical across jobs).

### rust-sdks prebuilts

The `rust_*` targets produce the prebuilt WebRTC that [livekit/rust-sdks](https://github.com/livekit/rust-sdks) `webrtc-sys` downloads, with the same names and layout as its `webrtc-builds` workflow: triples `linux-{x64,arm64}`, `android-{arm,arm64,x64}`, `mac-{x64,arm64}`, `ios-device-arm64`, `ios-simulator-arm64`, `win-{x64,arm64}`.

build.py only provides the checkout: it runs the build scripts in `webrtc-sys/libwebrtc` of the given rust-sdks checkout, which own the GN arguments, the patches and the checks on the result. The scripts build the synced commit in place of the branch pinned in their `.gclient`. `build_android.sh` needs JDK 17 or newer (Gradle 9).

These targets are not part of the release package set (`scripts/build.py packages`; `--all` lists them too). In CI they run on `platforms: all` and `rust`, may fail without failing the build, and upload `rust-packages-*` artifacts.

Set `GIT_CACHE_PATH` to reuse a gclient git cache across runs.

Test resources (`src/resources`, large raw audio/video downloaded from Google Storage) are only needed by `test`, which downloads and verifies them on every run. They are removed together with build outputs, so source caches never contain them.

Patches applied per target live in [scripts/patches](scripts/patches).

## Releasing

Releases are driven by pull requests:

1. Run the **release-pr** workflow with the new version (e.g. `150.7871.08`). It opens `release/<version>` with [VERSION](VERSION) pointing at the newest commit of the webrtc default branch (or a given commit), and the PR body lists the webrtc commits since the previous release. The PR's `build` checks produce every package.
2. **release-pr-refresh** keeps the PR current: on each webrtc push (`repository_dispatch: webrtc-updated`) and hourly it bumps `WEBRTC_COMMIT` to the newest commit and rewrites the commit list. Add the `pin` label to freeze the commit.
3. Merge the PR. **release** reuses the packages of the PR's last complete release build (or builds them again from the merge commit), checks that every package is there, creates the `m<version>` release with the commit list, the build inputs (this repository, libwebrtc wrapper), every package with its `.shasum` file, publishes LiveKitWebRTC, and notifies downstream repositories (`repository_dispatch: webrtc-release`).

Configuration:

| Name | Kind | Purpose |
|---|---|---|
| `RELEASE_APP_CLIENT_ID` | variable | Client ID of the release GitHub App. Its tokens push and open the release PRs, so the PRs trigger `build` (without the App: `GITHUB_TOKEN`, which does not), and send `repository_dispatch` to the downstream repositories (skipped without the App). |
| `RELEASE_APP_PRIVATE_KEY` | secret | Private key (`.pem`) of the release GitHub App. |
| `XCFRAMEWORK_PAT` | secret | Publishes LiveKitWebRTC to livekit/webrtc-xcframework. |
| `WEBRTC_REPO` | variable | webrtc repository (`owner/name`), default `webrtc-sdk/webrtc`. |
| `DOWNSTREAM_REPOS` | variable | Space-separated repositories to notify, default `webrtc-sdk/Specs webrtc-sdk/android`. They must share one owner, with the App installed on them. |
