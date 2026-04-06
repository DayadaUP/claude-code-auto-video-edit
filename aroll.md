---
description: "Auto rough-cut A-roll: Whisper transcribe → analyze edit points → generate DaVinci Resolve timeline + SRT subtitles"
---

# A-Roll Auto Rough Cut

Automate the tedious rough cut of talking-head / voiceover footage. Give it a folder of clips, and it will transcribe, detect keep/drop segments, and build a ready-to-use DaVinci Resolve timeline with synced subtitles.

## Input

$ARGUMENTS

If the user did not provide enough info, ask for:
1. **Footage folder path** (required)
2. **Landscape or portrait** (required) — landscape: 3840x2160 | portrait: 2160x3840, or let the user specify custom resolution
3. **Project name** (optional) — inferred from folder name if not provided
4. **Frame rate** (optional) — default 23.976. Common options: 23.976 / 24 / 25 / 29.97 / 30 / 50 / 59.94 / 60
5. **Language** (optional) — default auto-detect. Examples: Chinese, English, Japanese, etc.

## Workflow

### Step 1: Scan footage
- Scan the folder for video files (.MP4, .mp4, .MOV, .mov, .mxf, .MXF, .avi, .AVI)
- Sort by filename (usually contains shooting sequence number)
- List found files and confirm with the user before proceeding

### Step 2: Extract audio
- Use FFmpeg to extract 16kHz mono WAV from each video
- Command: `ffmpeg -i input.mp4 -vn -ar 16000 -ac 1 output.wav`

### Step 3: Transcribe with Whisper
- Transcribe each audio file using OpenAI Whisper with word-level timestamps
- Command: `whisper audio.wav --model medium --language {LANGUAGE} --word_timestamps True --output_format json`
- If language is set to auto-detect, omit the `--language` flag

### Step 4: Silence detection
- Use FFmpeg silencedetect to find silent gaps
- Command: `ffmpeg -i audio.wav -af silencedetect=noise={SILENCE_THRESHOLD}:d={MIN_SILENCE_DURATION} -f null -`
- Default threshold: -50dB, default minimum silence duration: 0.8s

### Step 5: Analyze edit points
Combine transcription and silence data to decide what to keep and what to cut:

**Keep/drop markers** — The creator may use spoken markers during recording to flag segments. Default markers:
- **Keep markers**: "OK", "好", "这条好", "keep", "good", "nice" (the segment *before* this marker is kept)
- **Drop markers**: "pass", "不要", "重来", "cut", "again", "redo" (the segment before this marker is dropped)

If no markers are detected in the entire session, treat all non-silent speech segments as kept (the creator didn't use a marker workflow).

**Cutting rules:**
- Remove silent gaps between speech segments
- Remove repeated takes of the same content — when multiple takes exist, keep the last good take (the one closest to a keep marker, or the last one if no markers)
- Trim leading/trailing dead air from each clip
- Collect all kept segments as `(source_filename, in_point_seconds, out_point_seconds)`

**After analysis, present the edit decision list to the user for review before proceeding.** Show: segment number, source file, in/out timecodes, first few words of transcript, and reason (kept/cut). Wait for user confirmation.

### Step 6: Generate DaVinci Resolve script
Generate a Python script (saved inside the footage folder) that uses the DaVinci Resolve scripting API to:
1. `pm.CreateProject()` — create a new project with the given name
2. `mp.AddSubFolder()` — create an `aroll` folder in the media pool root
3. `mp.SetCurrentFolder()` + `mp.ImportMedia()` — import source files into the aroll folder
4. `project.SetSetting()` — set resolution and frame rate
5. `mp.CreateEmptyTimeline()` — create a timeline
6. `mp.AppendToTimeline()` — add clips with subframe-accurate in/out points

The script must set these environment variables before importing DaVinci modules:
```
PYTHONPATH="/Library/Application Support/Blackmagic Design/DaVinci Resolve/Developer/Scripting/Modules/"
```
For macOS also set:
```
DYLD_LIBRARY_PATH="/Applications/DaVinci Resolve/DaVinci Resolve.app/Contents/Libraries/Fusion/"
```

> **Note for Windows/Linux users**: Adjust the paths above to match your DaVinci Resolve installation. The script includes comments showing where to change them.

### Step 7: Generate SRT subtitles
- Generate a merged SRT file based on the **timeline time** (record time), not source file time
- Timecodes must align to the edited timeline
- Save the .srt file in the footage folder

### Step 8: Execute
- Check if DaVinci Resolve is already running:
  - macOS: `pgrep -x "DaVinci Resolve"`
  - Windows: `tasklist /FI "IMAGENAME eq Resolve.exe"`
  - Linux: `pgrep -x resolve`
- If not running, launch it automatically:
  - macOS: `open -a "DaVinci Resolve"`
  - Windows/Linux: add a comment in the script showing the typical executable path for the user to configure
- Wait for the DaVinci Resolve scripting API to be ready. Add a retry loop at the start of the generated Python script:
  ```python
  import time
  resolve = None
  for i in range(30):  # wait up to 30 seconds
      try:
          import DaVinciResolveScript as dvr
          resolve = dvr.scriptapp("Resolve")
          if resolve:
              break
      except:
          pass
      time.sleep(1)
  if not resolve:
      print("Error: Could not connect to DaVinci Resolve. Please make sure it is fully launched.")
      exit(1)
  ```
- Run the generated Python script
- Verify the timeline was created successfully
- Report results and tell the user where the SRT file is

## Configuration Reference

These defaults can be adjusted per-run by telling the assistant your preferences:

| Parameter | Default | Description |
|-----------|---------|-------------|
| Resolution | 3840x2160 (landscape) | Output timeline resolution |
| Frame rate | 23.976 fps | Timeline frame rate |
| Language | auto-detect | Whisper transcription language |
| Whisper model | medium | Model size: tiny / base / small / medium / large |
| Silence threshold | -50 dB | Below this level counts as silence |
| Min silence duration | 0.8s | Minimum gap length to cut |
| Keep markers | OK, 好, keep, good | Spoken words that mark a good take |
| Drop markers | pass, 不要, cut, again | Spoken words that mark a bad take |

## Key Rules
- One folder = one project = one merged timeline + one SRT
- All source clips are laid out on a single timeline in shooting order
- Intermediate files (WAV, JSON) are kept until the user says to clean up
- Always show the edit decision list before executing — never auto-cut without confirmation
