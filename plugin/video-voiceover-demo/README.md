# video-voiceover-demo

Demo plugin for adding an AI voiceover to a marketing video that lives on Google Drive, without ever pushing video bytes through the model's context window.

## What's included

- **Skills**
  - `video-toolchain-check` — prerequisite check: ffmpeg/ffprobe present, Google Drive Desktop mounted as a local drive, HeyGen MCP authenticated. Run first in any new session.
  - `voiceover-inserter` — the actual editing workflow (see below).
- **Command**
  - `/add-voiceover [file] [script]` — triggers the workflow end to end.

## Workflow: Input → sample content → Output

```
Google Drive (Desktop, mounted e.g. as G:\My Drive\<Project>\)
  Input\  ──▶  1. Probe source video (ffprobe: duration, resolution, existing audio)
               2. Draft a short sample script (or use the user's script)
               3. Synthesize speech per segment (HeyGen TTS, engine=starfish)
               4. Download each speech clip locally (small file, direct HTTP)
               5. Mux into the video with ffmpeg: duck original audio under
                  each voice segment, keep video stream untouched (-c:v copy)
               6. (optional) Trim to a short/teaser cut ending right after
                  the voiceover finishes
  Output\ ◀──  7. Copy the versioned result in — Drive Desktop syncs it
               automatically, no API upload needed
```

Each exported file is named to carry its own version history, e.g. `Video Project 12 1_with_voice_v2.mp4`, `..._short.mp4` — so multiple iterations can sit side by side in `Output` and be compared without opening them.

## Why no Drive API for the video itself

Moving a video through a tool's base64 field costs roughly `(file_bytes × 1.33) / 2.5` tokens — a 50MB clip is ~25-30M tokens, i.e. it doesn't complete. Google Drive Desktop turns "download"/"upload" into a plain local file copy, which is the only approach this plugin uses for anything above a few MB. The Drive MCP tools are still useful for search/metadata/small text files, just not for moving the video itself.

## Requirements

- ffmpeg (the skill installs it via winget if missing, with confirmation)
- Google Drive for Desktop, signed in, with the target project folder synced
- HeyGen MCP connector authorized for the workspace (TTS + voice listing)

## Known limitations (demo scope)

- Ducking uses hardcoded time windows per segment, not sidechain compression — fine for short demo scripts, recalculate windows if script timing changes. See the "production version" note in `voiceover-inserter`.
- Assumes a single source video per run; batch processing across `Input` is not implemented.
- Assumes Windows + winget for the ffmpeg prerequisite check.
