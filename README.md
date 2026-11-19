![mpv logo](https://raw.githubusercontent.com/mpv-player/mpv.io/master/source/images/mpv-logo-128.png)
# mpv Omniphony iSpatialAudioClient

**A custom Windows Spatial Audio output backend for mpv, powered by Microsoft's `ISpatialAudioClient` API.**

[![Windows Spatial Audio Build](https://github.com/dfdfsdf-hub-audio/mpv-Omniphony-iSpatialAudioClient/actions/workflows/build-wasapi-spatial.yml/badge.svg)](https://github.com/dfdfsdf-hub-audio/mpv-Omniphony-iSpatialAudioClient/actions)

`mpv-Omniphony-iSpatialAudioClient` is a customized version of [mpv](https://github.com/mpv-player/mpv) that introduces a dedicated Windows WASAPI Spatial Audio output backend.

The project aims to provide a direct multichannel PCM output path through Microsoft's Windows Spatial Audio API, enabling compatible Windows spatial audio processing pipelines to receive multichannel audio from mpv.

The project is designed to work alongside external audio rendering software such as **Omniphony Studio**, allowing rendered multichannel audio to be passed into the Windows Spatial Audio pipeline.

---

## Contents

- [Project Overview](#project-overview)
- [Key Features](#key-features)
- [Audio Architecture](#audio-architecture)
- [Supported Platforms](#supported-platforms)
- [Building](#building)
- [Windows Spatial Audio Output](#windows-spatial-audio-output)
- [Omniphony Studio Integration](#omniphony-studio-integration)
- [Configuration Example](#configuration-example)
- [Build Status](#build-status)
- [Limitations](#limitations)
- [Development](#development)
- [Acknowledgements](#acknowledgements)
- [License](#license)

---

## Project Overview

The standard mpv audio output architecture provides several audio output backends, including WASAPI on Windows.

This project extends that architecture with a dedicated output backend:

```text
ao=wasapi-spatial
```

Unlike conventional WASAPI shared-mode output, the custom backend is designed around the Windows Spatial Audio API:

```text
ISpatialAudioClient
ISpatialAudioObjectRenderStream
ISpatialAudioObject
```

The implementation uses Windows event-driven spatial audio rendering to submit audio through spatial audio objects.

The main objective is to provide a multichannel PCM output backend that can participate in a Windows Spatial Audio processing chain.

## Key Features

### Windows Spatial Audio API

- Dedicated `wasapi-spatial` audio output backend.
- Uses Microsoft's `ISpatialAudioClient` interface.
- Event-driven rendering architecture.
- Spatial audio object buffer management.
- Multichannel PCM submission through spatial audio objects.

### Audio Rendering and Scheduling

- MMCSS audio thread scheduling.
- Attempts to register the rendering thread with the Windows Pro Audio task profile, with an Audio task fallback.
- High-priority rendering thread scheduling.
- QPC-based rendering timing measurements.
- Render-time diagnostics and slow-render event monitoring.
- Audio clock integration where supported, with fallback timing estimation.

### mpv Integration

- Integrated into mpv's audio output subsystem.
- Retains mpv's existing decoding and audio processing pipeline.
- Supports mpv's existing media formats and decoders, subject to the capabilities of the selected build.
- Designed to work with externally rendered multichannel PCM.

---

## Audio Architecture

The intended audio processing chain is:

```text
                 Media Sources
                       |
                       v
                mpv Demuxer
                       |
                       v
                Audio Decoder
                       |
                       v
             mpv Audio Processing
                       |
                       v
              Multichannel PCM
                       |
                       v
              ao=wasapi-spatial
                       |
                       v
             Windows Spatial Audio
                ISpatialAudioClient
                       |
                       v
           Windows Spatial Audio Pipeline
                       |
                       v
             Compatible Spatial Audio
                  Endpoint / Engine
```

For an external rendering workflow:

```text
        Dolby Atmos / Multichannel Sources
                       |
                       v
              Omniphony Studio
                       |
                       v
            Multichannel PCM Rendering
                       |
                       v
                  mpv Input
                       |
                       v
              WASAPI Spatial Backend
                       |
                       v
             Windows Spatial Audio
```

**Important:** This project provides a Windows Spatial Audio output backend. It does not independently decode Dolby Atmos bitstreams or replace an Atmos renderer. Atmos decoding and rendering depend on the upstream decoder or external rendering software used in the audio workflow.

---

## Supported Platforms

### Windows

The primary development and testing target is:

- Windows 10 version 1607 or later.
- Windows 11.
- Windows Spatial Audio-compatible audio endpoints and processing environments.

The backend requires Windows Spatial Audio API support and a compatible spatial audio configuration.

Linux and macOS are not targets of the custom `wasapi-spatial` backend. Other mpv functionality remains subject to the original project's platform support.

---

## Building

This project uses the Meson build system and Ninja.

The Windows CI build script is:

```text
ci/build-win32-spatial.ps1
```

### Build prerequisites

- Windows 10 or Windows 11.
- Git.
- Python.
- Meson.
- Ninja.
- A supported C compiler toolchain.
- Windows SDK.
- The dependencies required by mpv and its enabled subprojects.

### Build using the project script

From a suitable PowerShell environment:

```powershell
git clone https://github.com/dfdfsdf-hub-audio/mpv-Omniphony-iSpatialAudioClient.git

cd mpv-Omniphony-iSpatialAudioClient

powershell -ExecutionPolicy Bypass -File .\ci\build-win32-spatial.ps1
```

The project build configuration enables the relevant Windows components, including:

```text
-Dwasapi=enabled
-Dorender=enabled
-Dd3d11=enabled
-Dvulkan=enabled
```

The build script also builds the mpv executables and libmpv library, performs a basic executable smoke test, and prepares build artifacts.

### Manual Meson build

A basic Meson build can be configured with:

```powershell
meson setup build-spatial `
    --wrap-mode=forcefallback `
    -Ddefault_library=static `
    -Dlibmpv=true `
    -Dtests=true `
    -Dgpl=true `
    -Dwasapi=enabled `
    -Dorender=enabled

ninja -C build-spatial
```

For a performance-oriented Release build, use a separate build directory:

```powershell
meson setup build-spatial-release `
    --buildtype=release `
    --optimization=3 `
    --wrap-mode=forcefallback `
    -Ddefault_library=static `
    -Dlibmpv=true `
    -Dtests=true `
    -Dgpl=true `
    -Dwasapi=enabled `
    -Dorender=enabled

ninja -C build-spatial-release
```

The manual commands above are illustrative configurations. For the complete project-specific dependency and feature configuration, refer to `ci/build-win32-spatial.ps1`.

---

## Windows Spatial Audio Output

The custom audio output driver is:

```ini
ao=wasapi-spatial
```

It uses Windows Spatial Audio interfaces to manage the rendering stream and submit audio buffers.

The rendering process includes:

1. Initializing the Spatial Audio client.
2. Creating the spatial audio render stream.
3. Waiting for rendering events.
4. Beginning an audio update cycle.
5. Obtaining buffers for spatial audio objects.
6. Filling the buffers with audio data.
7. Ending the update cycle to submit the rendered audio.

The backend also collects timing information to help investigate scheduling and rendering performance.

### Rendering diagnostics

The implementation includes diagnostic information such as:

- Maximum available frame count.
- Actual frame count per rendering update.
- Rendering period.
- Rendering duration.
- Slow rendering events.
- Rendering wait timeouts.
- Audio buffer acquisition and submission failures.
- Audio clock failures and fallback usage.

These measurements are intended to help identify rendering problems. They should not automatically be interpreted as proof of audible dropouts or hardware output latency.

---

## Omniphony Studio Integration

This project is intended to complement external multichannel audio rendering software, including Omniphony Studio.

A typical workflow is:

```text
Source Audio
     |
     v
Omniphony Studio
     |
     v
Multichannel PCM
     |
     v
mpv
     |
     v
WASAPI Spatial Audio Backend
     |
     v
Windows Spatial Audio
```

For example, a rendered 7.1.4 PCM signal can be supplied to mpv, with the intended channel layout explicitly configured.

The actual channel mapping, spatial object assignment, endpoint compatibility, and final playback behavior must be verified on the target Windows system.

This project focuses on the output transport and rendering integration rather than performing the Atmos object rendering itself.

---

## Configuration Example

A basic mpv configuration:

```ini
# Windows Spatial Audio output
ao=wasapi-spatial

# Example: 7.1.4 channel layout
audio-channels=fl-fr-fc-lfe-bl-br-sl-sr-tfl-tfr-tbl-tbr

# Keep the source channel layout explicit when required
```

The example above describes a 12-channel 7.1.4 speaker layout.

Channel ordering must match the actual input PCM channel order and the backend's spatial object mapping. Do not assume that every external renderer uses the same ordering.

For other layouts, configure `audio-channels` to match the actual PCM channel arrangement.

---

## Build Status

The project has successfully completed a Windows x64 CI build.

The verified build pipeline includes:

- Windows x64 compilation.
- WASAPI Spatial Audio backend compilation.
- Meson test execution.
- Build artifact generation and upload.
- Basic executable smoke testing.

Latest known successful build:

- Commit: `0c9b479a0b44ac7acdb7c9a6cf7a1979a4c5dc88`
- Branch: `feature/wasapi-spatial-audio-orender`
- Workflow run: [GitHub Actions #43596002227](https://github.com/dfdfsdf-hub-audio/mpv-Omniphony-iSpatialAudioClient/actions/runs/43596002227)

A successful CI build confirms that the tested source revision compiles and passes the configured tests. It does not, by itself, establish successful real-device playback, correct channel mapping, or the absence of audio dropouts.

---

## Limitations

This project is under active development.

The following areas require validation on real Windows hardware:

- Actual Windows Spatial Audio endpoint compatibility.
- Full multichannel playback stability.
- 7.1.4 channel mapping accuracy.
- Side and rear channel separation.
- Spatial audio object positioning.
- Rendering latency and scheduling behavior.
- Long-duration playback reliability.
- Compatibility with different Windows Spatial Audio processing modes.

The project should be considered an experimental custom mpv audio output implementation until these areas have been comprehensively validated.

---

## Development

The main custom audio output implementation is located at:

```text
audio/out/ao_wasapi_spatial.c
```

The Windows build script is located at:

```text
ci/build-win32-spatial.ps1
```

The project is based on mpv and follows its existing build architecture and audio output subsystem design.

Contributions, bug reports, and technical discussions are welcome through the repository's GitHub issue tracker.

- [Issues](https://github.com/dfdfsdf-hub-audio/mpv-Omniphony-iSpatialAudioClient/issues)
- [Pull Requests](https://github.com/dfdfsdf-hub-audio/mpv-Omniphony-iSpatialAudioClient/pulls)
- [Actions](https://github.com/dfdfsdf-hub-audio/mpv-Omniphony-iSpatialAudioClient/actions)

---

## Acknowledgements

This project is built upon the work of the original mpv project and the wider open-source multimedia ecosystem.

Special thanks to:

- ChatGPT-6
- [mpv](https://github.com/mpv-player/mpv)
- [FFmpeg](https://ffmpeg.org/)
- [Microsoft Windows Spatial Audio API](https://learn.microsoft.com/en-us/windows/win32/api/spatialaudioclient/)
  
- The developers and contributors of the open-source multimedia libraries used by mpv.

---

## License

This project is based on mpv.

The original mpv licensing terms apply:

- GPLv2 or later by default.
- LGPLv2.1 or later when built with `-Dgpl=false`.

See the original [mpv Copyright file](https://github.com/mpv-player/mpv/blob/master/Copyright) and the individual component licenses for details.

