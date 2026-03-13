# Lilipad Plan

## Purpose

`lilipad` is a local, AES-managed tooling container built on Alpine Linux.

It exists so AES can run external media tooling in a consistent cross-platform environment instead of depending primarily on host-installed binaries.

Initial tools provided by `lilipad`:

- `ffmpeg`
- `ffprobe`
- `yt-dlp`

`yt-dlp` is the only video downloader target for this plan. `youtube-dl` compatibility is not required.

## Product direction

AES should always prefer `lilipad` for external tool execution.

AES manages `lilipad` itself:

- detect whether a supported container runtime is available
- ensure the `lilipad` image exists locally
- rebuild or refresh the image when AES expects a newer version
- execute tool commands through the container

Native host tools are fallback-only and should be used only when `lilipad` is unavailable or fails.

## Container model

- Base image: Alpine Linux
- Packaging style: local command-runner container, not a published app service
- Default tools installed in image:
  - `ffmpeg`
  - `yt-dlp`
  - `bash`
  - `ca-certificates`
- No published ports by default
- No long-lived network service required for the first version

Expected execution pattern:

```text
docker run --rm ... lilipad ffmpeg ...
docker run --rm ... lilipad yt-dlp ...
```

The same model should work with `podman` as a fallback runtime.

## Why this exists

The current codebase assumes direct native execution or local binary installation:

- `AES_Controls/Helpers/FFmpegLocator.cs` searches the app directory and host `PATH`
- `AES_Controls/Helpers/FFmpegManager.cs` installs and upgrades FFmpeg on the host
- `AES_Controls/Helpers/YtDlpManager.cs` downloads `yt-dlp` into the app directory
- `AES_Lacrima/Program.cs` mutates `PATH` so bundled binaries can be found

That model is harder to keep consistent across Windows, macOS, and Linux. `lilipad` shifts the preferred execution path into a controlled container runtime while preserving a native fallback when needed.

## Lifecycle ownership

AES should manage the full lifecycle of `lilipad`.

### Startup or first-use responsibilities

- detect `docker` first, then `podman`
- verify the runtime is callable from AES
- inspect whether the `lilipad` image exists
- compare the local image against the version expected by AES
- build or rebuild the image when needed

### Ongoing responsibilities

- invoke all supported external tools through `lilipad`
- mount only the directories required for each operation
- keep logs and command output usable by existing parsing logic
- report container readiness and failure states to the UI

## Runtime behavior

AES should use `lilipad` by default for:

- metadata extraction
- thumbnail generation
- media probing
- stream URL resolution via `yt-dlp`
- download and transcode workflows
- any future tool-backed media operations

Native execution should remain available only for fallback or recovery.

## Storage and mounts

`lilipad` should use AES-owned host-mounted directories instead of storing user data in the image.

Suggested layout:

- `.lilipad/cache`
- `.lilipad/work`
- `.lilipad/downloads`

Per-operation media or temporary paths can be mounted as needed.

Goals:

- persistent cache between runs
- ephemeral containers
- predictable file locations for AES
- avoid polluting the host with ad hoc binary installs

## Cross-platform expectations

The user experience should be consistent on:

- Windows
- macOS
- Linux

AES should construct and execute the container commands internally. Users should not need to manually run helper scripts during normal use.

## AES integration points

The current direct-binary model needs an execution abstraction so callers do not care whether a tool runs natively or inside `lilipad`.

Recommended new abstraction:

- `IToolExecutionBackend`
- `LilipadToolBackend`
- `NativeToolBackend`

Likely first consumers to migrate:

- `AES_Controls/Helpers/FFmpegLocator.cs`
- `AES_Controls/Helpers/FFmpegManager.cs`
- `AES_Controls/Helpers/YtDlpManager.cs`
- `AES_Lacrima/Services/MetadataService.cs`
- `AES_Lacrima/Services/MediaUrlService.cs`
- `AES_Controls/Player/AudioPlayer.cs`

## Manager behavior changes

### FFmpeg

`FFmpegManager` should stop acting primarily as a host installer.

Preferred-mode responsibilities:

- verify `lilipad` is available
- verify `ffmpeg` and `ffprobe` are callable inside the container
- expose status and error reporting for container readiness

### yt-dlp

`YtDlpManager` should stop treating app-directory binary downloads as the primary path.

Preferred-mode responsibilities:

- verify `lilipad` is available
- verify `yt-dlp` is callable inside the container
- report version and update state based on the managed image or tool version inside it

Existing local binary installation can remain as fallback behavior only.

## Versioning strategy

AES should own the expected `lilipad` version.

Possible approach:

- label the image with an AES-managed version string
- store the expected version in app code or config
- rebuild when the expected version does not match the local image

This lets AES migrate the tool environment intentionally as the app evolves.

## Build source

`lilipad` should be built locally from a repo-managed Dockerfile by default.

Rationale:

- keeps behavior tied to the checked-out codebase
- keeps tool/runtime definition versioned with AES
- avoids requiring a separate registry dependency for the first version

## Fallback policy

`lilipad` is preferred at all times, but native fallback remains necessary when:

- no supported container runtime is installed
- the runtime is installed but inaccessible
- the `lilipad` image fails to build
- a containerized tool invocation fails and AES can recover natively

Fallback should be explicit in logs and UI status so users know when AES is no longer using the managed container path.

## Proposed implementation phases

1. Add this planning document.
2. Add a repo-managed Alpine Dockerfile and entrypoint for `lilipad`.
3. Add AES runtime detection, image inspection, and image build logic.
4. Add a tool execution abstraction and implement `lilipad` plus native fallback backends.
5. Migrate FFmpeg and `yt-dlp` call sites to the new backend.
6. Add UI/status reporting for runtime availability, image readiness, and failure modes.
7. Expand `BUILDING.md` with local setup and troubleshooting notes.

## Success criteria

- AES can build and run `lilipad` locally on Windows, macOS, and Linux.
- AES prefers `lilipad` automatically when a supported container runtime is available.
- `ffmpeg`, `ffprobe`, and `yt-dlp` can be invoked through the managed container.
- Existing media flows continue to work without requiring users to install host binaries first.
- Native fallback remains possible when container execution is unavailable.
