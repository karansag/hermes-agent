---
name: narrated-manim-video
description: "End-to-end workflow for producing a fully packaged narrated Manim explainer video: gather topic/depth/style requirements, write the scene plan and script, render scenes with subcaptions, create stitched subtitles, generate TTS voiceover per scene, align narration timing, and deliver a final narrated MP4."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [manim, video, narration, captions, tts, explainer]
    related_skills: [manim-video]
    category: creative
---

# Narrated Manim Video

Use this when the user wants a Manim explainer video and expects the whole pipeline completed, not just scene code.

This skill extends `manim-video` with:
- upfront requirement gathering for scope and tone
- scene-appropriate subcaptions in code
- stitched `final.srt` for the stitched video
- per-scene narration generation
- per-scene timing alignment of narration to scene durations
- final packaged narrated MP4 deliverable

Default deliverable:
- `final-narrated-aligned.mp4`

Secondary deliverables:
- `final.mp4`
- `final.srt`
- per-scene scene videos and `.srt`
- per-scene narration audio

## Load This Skill With
Also load `manim-video` and follow its visual/creative standards. This skill is the packaging and narration layer on top.

## First: gather the right video requirements
When the user says “make a video about X”, do not jump straight into code. Ask for a concise brief tailored to the topic.

Gather:
1. topic / exact question the video should answer
2. audience level
   - beginner
   - intermediate
   - advanced
3. desired depth
   - short intuition
   - medium explainer
   - deeper walkthrough
4. preferred style
   - geometric intuition
   - equation-first
   - algorithm walkthrough
   - architecture/system explanation
5. duration target if they care
6. whether they want narration, captions, or both
   - default: both
7. whether they want neutral/professional, energetic, or conversational narration tone
8. any must-cover or must-avoid points

If the request is underspecified, ask a compact question set before proceeding.

Suggested prompt:
- What exact topic/question should the video answer?
- Who is it for: beginner, intermediate, or advanced?
- Do you want short intuition, medium depth, or a deeper walkthrough?
- Should the video emphasize geometry/intuition, equations, algorithms, or systems diagrams?
- Any must-cover points or tone preferences?

## Project setup
Prefer a dedicated project directory with uv.

Example:
```bash
mkdir -p project-name
cd project-name
uv init --bare --python 3.11
uv add manim requests
```

System prerequisites on Ubuntu/Debian:
```bash
sudo apt-get update
sudo apt-get install -y pkg-config libcairo2-dev libpango1.0-dev
```

Also required:
- `ffmpeg`
- `pdflatex` / LaTeX

Verify:
```bash
uv run manim --version
ffmpeg -version
pdflatex --version
```

## Required project artifacts
Keep the project simple:

```text
project-name/
  plan.md
  script.py
  concat.txt
  final.mp4
  final.srt
  final-narrated-aligned.mp4
  narration/
    Scene1_....txt
  audio/
    Scene1_....mp3
    Scene1_....-aligned.mp3
    full-narration.mp3
    full-narration-aligned.mp3
  media/
```

## Workflow overview
1. clarify the video brief
2. write `plan.md`
3. write `script.py` with one scene class per scene
4. include `self.add_subcaption(...)` lines in each scene
5. render scene clips with Manim
6. stitch scene clips into `final.mp4`
7. combine per-scene `.srt` files into `final.srt`
8. generate per-scene narration text from the subtitle copy
9. synthesize per-scene audio with TTS
10. align per-scene audio durations to the actual scene durations
11. concatenate aligned scene audio into `full-narration-aligned.mp3`
12. mux aligned narration with `final.mp4` into `final-narrated-aligned.mp4`
13. verify streams and deliver the final artifact

## Script requirements
In `script.py`:
- one scene class per scene
- every scene independently renderable
- `self.camera.background_color = ...` set per scene
- shared palette constants at the top
- use monospace fonts for `Text(...)`
- add subcaptions throughout
- clean scene exits with fadeouts

Subcaption rule:
- write scene-appropriate spoken lines, not just labels
- the text should sound natural enough for both subtitles and narration
- think of each `self.add_subcaption(...)` line as a draft narration script

Example:
```python
self.add_subcaption("The real reason is geometric.", duration=2)
self.play(Write(title), run_time=1.3)
self.wait(0.8)
```

## Rendering scene clips
Draft render first:
```bash
uv run manim -ql script.py Scene1 Scene2 Scene3 Scene4
```

Stitch scene clips:
```bash
ffmpeg -y -f concat -safe 0 -i concat.txt -c copy final.mp4
```

Verify stitched video:
```bash
ffprobe -v error -show_entries format=duration:stream=index,codec_type,codec_name,width,height -of json final.mp4
```

## Build stitched subtitles for the final video
Important: Manim generates per-scene `.srt` files, not a final stitched subtitle file.

You must create `final.srt` by:
1. reading the scene list from `concat.txt` (or using the same ordered scene list)
2. measuring each scene video duration with `ffprobe`
3. offsetting each scene subtitle block by the cumulative durations of prior scenes
4. writing one combined `final.srt`

This is necessary because per-scene subtitle timestamps start at zero for each scene.

Verification:
- `final.srt` should span approximately the same duration as `final.mp4`
- players can then use `final.mp4` + `final.srt`

## TTS workflow
Recommended default provider for low-cost narration: xAI TTS.

Credential handling:
- Prefer environment variables such as `XAI_API_KEY` in `~/.hermes/.env` or the shell environment.
- Do not commit API keys, tokens, `.env` files, or project-local secret files into a repo.
- If you need to mention example configuration in generated project docs, always use placeholder values only.

Example placeholder configuration:
```text
XAI_API_KEY=your_key_here
```

If a workflow already uses `XAI_API_TOKEN`, that can be adapted to the same request pattern.

Basic xAI TTS request:
```python
import json, urllib.request

payload = json.dumps({
    "text": text,
    "voice_id": "rex",
    "language": "en",
    "text_normalization": True,
    "output_format": {
        "codec": "mp3",
        "sample_rate": 44100,
        "bit_rate": 192000,
    },
}).encode("utf-8")

req = urllib.request.Request(
    "https://api.x.ai/v1/tts",
    data=payload,
    headers={
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    },
    method="POST",
)
```

Suggested default voice:
- `rex` for clear instructional/professional delivery

Alternatives:
- `ara` for warmer narration
- `eve` for more energetic narration

## Generate narration text per scene
Use the subtitle copy as the base narration script.

Process:
1. read each scene `.srt`
2. extract only the spoken text lines
3. join blocks with short pauses
4. write one text file per scene in `narration/`

Example join strategy:
```text
Sentence 1. [pause] Sentence 2. [pause] Sentence 3.
```

Why per-scene narration is better than one giant narration blob:
- easier to regenerate only one section
- easier to align timings scene by scene
- matches the scene-based structure of Manim

## Timing alignment step is mandatory
Raw TTS duration often will not match scene duration.
Do not skip alignment.

Procedure:
1. measure scene video duration with `ffprobe`
2. measure generated scene audio duration with `ffprobe`
3. compute ratio = audio_duration / video_duration
4. run ffmpeg `atempo=ratio` to retime the audio toward the video duration
5. write `SceneX-aligned.mp3`
6. concatenate aligned scene audio files

Example command pattern:
```bash
ffmpeg -y -i Scene1.mp3 -filter:a "atempo=1.23" -vn Scene1-aligned.mp3
```

Then concatenate aligned audio:
```bash
ffmpeg -y -f concat -safe 0 -i audio_concat_aligned.txt -c copy audio/full-narration-aligned.mp3
```

Then mux:
```bash
ffmpeg -y -i final.mp4 -i audio/full-narration-aligned.mp3 -c:v copy -c:a aac -b:a 192k -shortest final-narrated-aligned.mp4
```

## Verification checklist
Before finishing, verify all of these with tools:

1. `final.mp4` exists
2. per-scene `.srt` files exist
3. `final.srt` exists
4. per-scene audio files exist
5. `full-narration-aligned.mp3` exists
6. `final-narrated-aligned.mp4` exists
7. `ffprobe` on final narrated video shows:
   - one video stream
   - one audio stream
8. total narrated video duration matches the video runtime
9. if possible, spot-check one subtitle file and one final stream probe

## Delivery behavior
Default final deliverable to point the user at:
- `final-narrated-aligned.mp4`

Also mention:
- `final.mp4` for silent version
- `final.srt` for captions as a sidecar subtitle file

Playback examples:
```bash
ffplay final-narrated-aligned.mp4
ffplay -i final-narrated-aligned.mp4 -vf subtitles=final.srt
```

## Common pitfalls
- assuming scene `.srt` files will automatically work with stitched `final.mp4`
- generating one long narration file without scene-level alignment
- forgetting that TTS audio may be significantly longer than scene durations
- checking secrets into a repository or embedding them in example files
- trusting redacted key previews when debugging API auth
- treating subcaptions as throwaway labels instead of spoken narration copy
- delivering only code when the user expects a final packaged artifact

## Default decision policy
If the user does not specify otherwise:
- make a medium-depth explainer
- include both narration and captions
- use xAI TTS with `rex`
- produce a final packaged narrated MP4
- keep the silent `final.mp4` and `final.srt` as secondary artifacts

## Completion template
When done, report:
- what topic the video covers
- main output file: `final-narrated-aligned.mp4`
- secondary files: `final.mp4`, `final.srt`
- whether narration was generated and with which voice
- any timing caveats or if additional polish might help
