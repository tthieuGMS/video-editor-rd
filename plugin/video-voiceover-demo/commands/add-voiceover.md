---
description: Add a sample AI voiceover to a video from the project's Google Drive Input folder and export the result to Output.
---

Run the `video-toolchain-check` skill, then the `voiceover-inserter` skill against the video the user names (or the most recent file in the project's `Input` folder if none is named). If the user gave a script, use it verbatim; otherwise generate a short sample voiceover per the skill's guidance. Export the result to `Output` and report the exported filename, duration, and what each version contains.

Arguments: $ARGUMENTS (optional — a filename in Input, and/or a script to narrate)
