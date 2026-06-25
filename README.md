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

## Notes

- Do not use GitHub's "Sync fork" button unless you intentionally want to leave
  the v20.14.0 maintenance line.
- Upstream ALVR documentation and general project information live at
  <https://github.com/alvr-org/ALVR>.
- This fork is for a specific v20.14.0 Quest 3 workflow, not a replacement for
  upstream ALVR releases.

## License

ALVR is licensed under the [MIT License](LICENSE).
