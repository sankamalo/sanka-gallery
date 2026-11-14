---
name: reels-editor
description: Edit Instagram Reels from raw footage with ffmpeg — cut/trim, join clips, reframe to 9:16, add burned-in subtitles, add text overlays, music and speed changes, matching a reference video's style. Use when the user sends raw footage (and optionally a reference reel) and asks to cut, subtitle, add text or assemble a reel.
---

# Reels editor (ffmpeg)

Tools: `ffmpeg`/`ffprobe` with `drawtext`, `subtitles` and `ass` (libass). Fonts: Inter, Inter Display (Black/ExtraBold for hooks), DejaVu Sans.

## Workflow
1. **Inputs** go in `media/in/` (raw = `bruto*`, reference = `ref*`). Outputs in `media/out/`. Large videos are NOT committed (see `.gitignore`).
1b. **Instagram reference link**: `pip install -q yt-dlp` then `yt-dlp -o media/in/ref.%(ext)s <URL>`. Needs the `sanka-gallery` cloud environment (allows instagram.com). If the download fails with proxy 403, run `yt-dlp -g <URL>` to see the video CDN host (e.g. `scontent-xxx-1.cdninstagram.com`) and ask to add that exact host with update_environment.
2. **Inspect**: `ffprobe -v error -show_entries format=duration:stream=width,height,r_frame_rate,codec_name -of json FILE`.
3. **Study the reference** (if given): extract frames `ffmpeg -i ref.mp4 -vf fps=1,scale=360:-1 media/tmp/ref_%03d.jpg` and look at them: cut rhythm (count scene changes with `-vf "select='gt(scene,0.3)',showinfo"`), subtitle style (font, size, colour, position, words per line), text overlays, duration, music.
4. **Look at the raw footage** the same way, plus audio. Propose a short edit plan (timestamps for each cut + texts) and confirm with the user before rendering if anything is ambiguous.
5. **Render**, then extract a few frames of the result and check them before delivering.

## Recipes
- Trim: `ffmpeg -ss 00:00:03.2 -to 00:00:09.8 -i in.mp4 -c:v libx264 -crf 18 -preset fast -c:a aac clip1.mp4`
- Join (same format): list file with `file 'clip1.mp4'` lines → `ffmpeg -f concat -safe 0 -i list.txt -c copy joined.mp4`
- 9:16 1080x1920 crop-fill: `-vf "scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,setsar=1"`
- Text overlay (hook, first 3 s):
  `-vf "drawtext=fontfile=/path/Inter-Black.ttf:text='TEXTO':fontsize=86:fontcolor=white:borderw=6:bordercolor=black:x=(w-tw)/2:y=h*0.18:enable='between(t,0,3)'"`
  (find font paths with `fc-list | grep -i inter`; escape `:` and `'` in text, or use `textfile=`).
- Subtitles: write an `.ass` file (style: Inter Display Black, ~80px, white, black outline 5, Alignment=2, MarginV≈500 so they sit above Instagram UI), 1–4 words per line, then `-vf "ass=subs.ass"`.
  Timings: transcribe the audio (try `faster-whisper` with word timestamps; if no model can be downloaded, ask the user for the script and align manually by listening to silence gaps with `silencedetect`).
- Speed: `-filter:v setpts=0.8*PTS -filter:a atempo=1.25`
- Music under voice: `-filter_complex "[1:a]volume=0.15[m];[0:a][m]amix=inputs=2:duration=first"`
- Final export for Instagram: H.264 High, `-pix_fmt yuv420p -r 30 -b:v 8M -c:a aac -b:a 192k -ar 48000 -movflags +faststart`, 1080x1920, ≤ 90 s.

## Safe zones (Reels)
Keep key text between ~250 px from top and ~420 px from bottom; avoid the right 140 px (buttons).

## Delivery
Send the final file to the user with SendUserFile (and upload to Google Drive if they ask). Don't commit videos to git.
