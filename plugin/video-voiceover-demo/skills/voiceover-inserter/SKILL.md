---
name: voiceover-inserter
description: End-to-end workflow to add an AI voiceover to a video sitting in a Google Drive Input folder and export a versioned result to the matching Output folder. Use when the user asks to add/insert voice, narration, or a voiceover into a video that lives on Google Drive. Requires video-toolchain-check to have passed first.
metadata:
  type: workflow
---

# Voiceover inserter

Adds one or more spoken segments to an existing video and writes the result to `Output` next to the project's `Input` folder, without ever pushing video bytes through the model context.

Run [[video-toolchain-check]] first in any new session — this skill assumes ffmpeg's full path, the Drive Desktop mount, and HeyGen auth are already confirmed.

## Step 1 — Locate the source video in Input

Resolve the project folder under the Drive mount (e.g. `G:\My Drive\<Project>\Input`) and list it to find the target file — don't guess the filename. Probe it before touching anything:

```powershell
& "<ffprobe-path>" -v error -show_entries "format=duration,size:stream=codec_type,codec_name,width,height,channels" -of default=noprint_wrappers=0 "<input-file>"
```

Record duration, resolution, and whether an audio track already exists — the mux step in Step 4 depends on all three.

## Step 2 — Generate the sample voiceover script

If the user hasn't supplied a script, write one yourself — ask only what's actually ambiguous (voiceover-over-background vs. lipsync; language) rather than blocking on content details. A short marketing-style VO in the video's apparent language is a reasonable default for a "sample" ask.

Keep each segment under ~15-20s (roughly 35-45 words at natural pace) — long single segments are harder to re-time if the user asks for edits later. Multiple short segments compose more easily than one long one.

## Step 3 — Synthesize speech (HeyGen TTS)

1. Pick a voice: call the HeyGen list-voices tool with `engine=starfish` and the target `language`. Prefer a same-language public voice over a multilingual one when available.
2. Call the HeyGen create-speech tool with the script text and chosen `voiceId`. It returns an `audio_url` and the actual spoken `duration` — use the real duration, not your word-count estimate, for placement math in Step 4.
3. Download the audio straight to disk with a direct HTTP request (`Invoke-WebRequest -OutFile`) — these clips are small (hundreds of KB), so this step alone is safe to do outside the local-file-only rule, but still no need to route it through the model.

## Step 4 — Mux into the video

Default approach — duck the original audio under each voice segment, mix, keep video untouched:

```powershell
& "<ffmpeg-path>" -y -i "<input.mp4>" -i "<voice1.wav>" [-i "<voiceN.wav>" ...] `
  -filter_complex "[0:a]volume='between(t,S1,E1)+between(t,S2,E2)+...':eval=frame[bg];[1:a]aformat=sample_rates=48000:channel_layouts=stereo,adelay=<S1*1000>|<S1*1000>[v1];...;[bg][v1][vN]amix=inputs=<N+1>:duration=first:normalize=0[aout]" `
  -map 0:v -map "[aout]" -c:v copy -c:a aac -b:a 192k "<output.mp4>"
```

- `-c:v copy` keeps video quality/size untouched — only re-encode audio.
- `Sn`/`En` are each segment's start/end in seconds; leave a small gap (~1s) between segments so ducking doesn't sound clipped.
- For a production version (not a quick demo), prefer `sidechaincompress` keyed off the voice track instead of hardcoded `between(t,...)` windows — it ducks based on actual voice presence and survives script-length edits without recalculating timestamps. Reach for the simpler time-window version only for fast demos.

If the user wants a short/teaser cut instead of the full video, trim after muxing rather than before — re-encode just that clip:

```powershell
& "<ffmpeg-path>" -y -i "<muxed.mp4>" -t <end_of_last_voice_segment_plus_buffer> -c:v libx264 -preset veryfast -crf 18 -c:a copy "<output_short.mp4>"
```

## Step 5 — Export a version to Output

Copy — don't move — the result into the project's `Output` folder on the Drive mount, using a name that encodes what changed:

```powershell
Copy-Item "<local-working-dir>\<name>_with_voice_v<N>.mp4" "G:\My Drive\<Project>\Output\<name>_with_voice_v<N>.mp4"
```

Google Drive Desktop syncs it automatically — there is nothing further to push through Drive's API. Report the exported filename, its duration, and a one-line description of what's in each version so the user can tell versions apart later.

## What NOT to do

- Don't call the Drive MCP tool's `create_file`/`download_file_content` with a video's `base64Content` — see [[base64-context-budget]] in video-toolchain-check. A 50MB video is roughly 25-30M tokens of base64 and will not complete.
- Don't re-encode video unless you're actually changing pixels — always `-c:v copy` when only audio changes.
- Don't hardcode `G:` — confirm the actual drive letter/mount each session; it's user-configurable in Google Drive Desktop settings.
