---
name: video-toolchain-check
description: Verify the local environment can edit video before running any voiceover/mux workflow — checks ffmpeg/ffprobe on PATH, the Google Drive Desktop mount, and HeyGen MCP auth. Use BEFORE voiceover-inserter, or whenever a video-edit command fails with a missing-tool error (ffmpeg not recognized, no G:\ drive, HeyGen 401).
metadata:
  type: prerequisite-check
---

# Video toolchain check

Run this before any video-editing task. It answers three questions: can we encode video, can we reach Drive without pushing bytes through the model context, and can we generate voice.

## 1. ffmpeg / ffprobe

Windows winget installs do not add ffmpeg to PATH until the shell restarts, so locate the binaries directly instead of assuming `ffmpeg` resolves:

```powershell
Get-ChildItem -Path "$env:LOCALAPPDATA\Microsoft\WinGet\Packages" -Filter "ffmpeg.exe" -Recurse -ErrorAction SilentlyContinue |
  Select-Object -First 1 -ExpandProperty DirectoryName
```

- If nothing is found: `winget install --id Gyan.FFmpeg -e --accept-package-agreements --accept-source-agreements` (confirm with the user first — it modifies the system).
- Once found, always call the **full path** to `ffmpeg.exe` / `ffprobe.exe` in every subsequent command in this session rather than the bare command name.

## 2. Google Drive Desktop mount

Never move a video file through a tool's base64 parameter — see [[base64-context-budget]] below. Instead confirm Drive is mounted as a local drive:

```powershell
Get-PSDrive -PSProvider FileSystem | Where-Object { $_.Description -like "*Google Drive*" } | Select-Object Name, Root
```

- Expect a drive (commonly `G:`) with root containing `My Drive`.
- If missing, tell the user to install/open Google Drive for Desktop and confirm the drive letter — do not fall back to the Drive MCP tool's base64 upload/download for anything above a few MB (see budget note).
- Resolve the project's Input/Output folders as plain paths under that mount, e.g. `G:\My Drive\<Project>\Input` and `...\Output`. List them with a normal directory listing to confirm names before writing anything.

## 3. HeyGen voice generation

Confirm the HeyGen MCP connection is authenticated (not the `hyperframes` CLI's separate `~/.heygen` credential — these are independent):

- Call the HeyGen "get current user" tool. A response with `email`/`subscription` confirms the MCP session is authenticated and ready for TTS calls.
- If it errors, HeyGen MCP needs to be (re)authorized via the client's connector settings — this cannot be done from a non-interactive CLI session; tell the user and stop.

## base64-context-budget

Rule of thumb used across this toolkit: `token_cost ≈ (file_bytes × 1.33) / 2.5`. Files under ~1-3MB (short audio/images) are fine to move through a tool's base64 field. Anything larger — and every video — goes through the local filesystem (ffmpeg + the Drive Desktop mount) instead, never through base64.
