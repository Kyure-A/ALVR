# ALVR v20.14.0 Quest 3 Server Fork

This fork is pinned to **ALVR v20.14.0** for compatibility with an existing
Quest 3 client APK. It intentionally does not track upstream `master`.

The purpose of this branch is narrow:

- keep the v20.14.0 server/client protocol intact
- avoid rebuilding or replacing the Quest 3 APK
- backport only server-side fixes needed for this setup
- improve Windows virtual microphone handling without pulling in newer ALVR protocol changes

## Included Fix

This fork adds a Windows server-side microphone stability fix for the v20.14.0
streamer. It improves virtual microphone device matching for VAC, VB-CABLE, and
VoiceMeeter, avoids treating microphone initialization failures as fatal
handshake errors, retries microphone initialization while streaming, and logs the
selected microphone sink/source devices.

The fix targets issues like ALVR 20.14.0 failing to start a stream when
microphone support is enabled and a virtual microphone pair cannot be resolved
cleanly.

## TODO

This fork aims for "almost completely stable" Windows audio/microphone behavior
while staying on the v20.14.0 server protocol. The planned work is ordered from
lowest-risk / highest-value to larger architectural backports.

Because local development is on macOS but the target is Windows, **CI/CD must be
working before the remaining audio fixes are validated**. The first item below is
the current priority.

- [x] **Modernize GitHub Actions release workflow**: Update deprecated actions in
      `.github/workflows/prepare-release.yml` and `.github/workflows/rust.yml`,
      specialize the release for Windows-only server builds, and make the
      workflow manually triggerable so Windows binaries can be built and released
      without a local Windows environment.
- [ ] **Backport upstream #2918**: Remove audio device enumeration from the
      dashboard headset speaker dropdown. This avoids slow/broken hardware
      queries that can block startup and contribute to handshake timeouts.
- [ ] **Fix game-audio retry busy-loop**: Sleep `RETRY_CONNECT_MIN_INTERVAL`
      before `continue` when `get_windows_device_id()` fails, so a missing
      device does not spin the thread.
- [ ] **Strengthen audio error logging**: Log errors from
      `receive_samples_loop()`, `sender.send()`, and microphone initialization
      retries instead of silently swallowing them with `.ok()`.
- [ ] **Exponential backoff for audio retries**: Replace the fixed 1-second
      retry interval in the microphone and game-audio loops with exponential
      backoff (capped) for faster recovery and less resource use.
- [ ] **Clean up Windows COM initialization**: Call `CoInitializeEx` once per
      audio thread and document why, instead of calling it on every device
      lookup.
- [ ] **Defer microphone device resolution until after handshake**: Resolve the
      virtual microphone pair only inside the microphone thread, never during
      the connection handshake, to eliminate device-enumeration timeouts from
      the handshake path.
- [ ] **Cache audio device enumeration**: Cache the cpal output/input device
      lists with a short TTL so retries do not repeatedly pay the enumeration
      cost that can be slow on cpal 0.15.
- [ ] **Windows device-ID based comparison**: Keep the `AudioDevice` wrapper but
      compare Windows endpoints by their WASAPI ID in `is_same_device()`,
      avoiding false matches caused by duplicate/localized device names.
- [ ] **Backport upstream #2944 (cpal 0.16 + rodio 0.21)**: This is the
      upstream fix for the cpal 0.15 device-enumeration slowness that causes
      "OS error 11" / handshake aborts. It removes `AudioDevice`, switches to
      device-ID comparison, and updates the rodio playback API. Requires broad
      testing.
- [ ] **Review default microphone preset**: Decide whether VAC should remain the
      default or whether the default should better match the documented setup
      for VB-CABLE/VoiceMeeter users.
- [ ] **Windows audio endpoint change notifications (optional)**: Implement
      `IMMNotificationClient` so the server can react to default device
      changes, plug/unplug, and disabled endpoints without waiting for the
      stream to fail first.

## Release

Because local development is done on macOS but the target is Windows, use the
GitHub Actions workflow to build and release Windows binaries.

1. Go to **Actions > Create release** in this repository.
2. Click **Run workflow**.
3. Optionally enter a version (e.g., `20.14.0-q3fix.1`). If left blank, `cargo xtask bump` will pick the next version automatically.
4. The workflow will:
   - Bump the version and commit the change.
   - Build the Windows streamer and launcher.
   - Create a **draft** release named `ALVR v<version> Windows Server Only (Quest 3 v20.14.0 APK compatible)`.
   - Upload `alvr_streamer_windows.zip` and `alvr_launcher_windows.zip`.
5. Review the draft release, then publish it manually.

This workflow builds **server-side Windows artifacts only**. It does not rebuild
the Quest 3 client APK, which matches this fork's goal of keeping the existing
v20.14.0 client intact.

## Build

### Windows prerequisites

The upstream build scripts expect a Windows development environment with
Chocolatey available. Install Chocolatey before running the ALVR dependency/build
steps, then use the regular ALVR Windows build setup for Visual Studio Build
Tools, Rust, and the native dependencies.

Build the streamer from this pinned branch:

```powershell
cargo xtask build-streamer --release
```

The Windows portable output is expected under:

```text
build\alvr_streamer_windows
```

Register that directory as the SteamVR driver root, not `target\release`:

```powershell
vrpathreg.exe adddriver "C:\path\to\ALVR\build\alvr_streamer_windows"
```

Useful sanity checks:

```powershell
Test-Path .\build\alvr_streamer_windows\driver.vrdrivermanifest
Test-Path .\build\alvr_streamer_windows\bin\win64\driver_alvr_server.dll
Test-Path ".\build\alvr_streamer_windows\ALVR Dashboard.exe"
vrpathreg.exe show
```

`vrpathreg.exe show` should list `alvr_server` with the
`build\alvr_streamer_windows` path. If it shows `[ NO DRIVER NAME FOUND ]` for
`target\release`, remove that bad registration and add the build output instead:

```powershell
vrpathreg.exe removedriver "C:\path\to\ALVR\target\release"
vrpathreg.exe adddriver "C:\path\to\ALVR\build\alvr_streamer_windows"
```

Do not copy or register `target\release` directly. Cargo may place
`alvr_server_openvr.dll` there, but `xtask build-streamer` packages it as:

```text
build\alvr_streamer_windows\bin\win64\driver_alvr_server.dll
```

## License

ALVR is licensed under the [MIT License](LICENSE).
