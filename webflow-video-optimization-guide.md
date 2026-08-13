# Webflow Video Optimization — Dev Guide

**Goal:** Make Webflow pages with video load fast. Get a 20–30 MB Full-HD video down to ~1–3 MB with no visible quality loss for a hero/background video.

**The key idea:** There is no magic "video hosting" doing this on our Next.js site. The small file size comes almost 100% from **how the video is encoded** (codec + bitrate + resolution) plus **lazy loading**. We can do the exact same thing on Webflow. This guide shows how.

**Do NOT** use Webflow's built-in "Background Video" element or drag a raw video into the Assets panel. Webflow re-encodes it badly and gives you zero control over size. We encode the video ourselves, host it on a CDN, and embed it with a plain `<video>` tag.

---

## The workflow (3 steps)

1. **Encode** the video locally with FFmpeg (or HandBrake) → small MP4 (+ optional WebM).
2. **Upload** the encoded files + a poster image to our CDN.
3. **Embed** with a `<video>` tag inside a Webflow Embed element, with lazy loading and a poster.

---

## Step 1 — Encode the video

Install FFmpeg once:
- Mac: `brew install ffmpeg`
- Windows: download from ffmpeg.org, or `winget install ffmpeg`
- Prefer a GUI? Use **HandBrake** (free) with the settings in the "HandBrake" section below.

### Before you encode: shrink 3 things
These three give 90% of the savings:

1. **Resolution.** A hero background sits behind text and overlays. It almost never needs true 1920px. Encode at **1280px or 1440px wide** — the difference is invisible, the file is much smaller. Only go 1920 if the video is sharp, front-and-center content.
2. **Frame rate.** If the source is 60fps, drop to **30fps** (or 24 for a cinematic feel). Half the frames = much smaller file.
3. **Audio.** Background/hero videos are muted anyway. **Remove the audio track** entirely (`-an`).

### Command A — Hero / background video (muted, autoplay, loop)

This is the main one. Strips audio, scales to 1440px wide, web-optimized:

```bash
ffmpeg -i input.mov \
  -c:v libx264 -crf 28 -preset slow \
  -vf "scale=1440:-2,fps=30" \
  -an \
  -movflags +faststart -pix_fmt yuv420p \
  output.mp4
```

What each part does:
- `-crf 28` → quality/size dial. **Lower = better quality + bigger; higher = smaller.** Sweet spot is **26–30**. Start at 28. If it looks soft, try 26. If size is still too big, try 30.
- `-preset slow` → encoder works harder for a smaller file. `slower` = even smaller but takes longer. Fine to use `slow`.
- `scale=1440:-2` → 1440px wide, height auto (the `-2` keeps it even-numbered, which H.264 requires).
- `fps=30` → force 30fps. Remove this line if the source is already 24/30.
- `-an` → remove audio.
- `-movflags +faststart` → **critical for web.** Moves the file index to the front so the video starts playing before it's fully downloaded. Never skip this.
- `-pix_fmt yuv420p` → guarantees it plays in all browsers (especially Safari/iOS).

### Command B — Video with sound (testimonial, product clip, etc.)

Same idea, but keep audio at a low bitrate:

```bash
ffmpeg -i input.mov \
  -c:v libx264 -crf 26 -preset slow \
  -vf "scale=1920:-2" \
  -c:a aac -b:a 128k \
  -movflags +faststart -pix_fmt yuv420p \
  output.mp4
```

### Command C — Optional WebM (VP9) for extra savings

VP9 in a WebM is usually **20–40% smaller** than H.264 at the same quality. Modern browsers use the WebM; older ones (and some Safari cases) fall back to the MP4. Encoding is slower, so only do this for the hero and other heavy videos, not every clip.

```bash
ffmpeg -i input.mov \
  -c:v libvpx-vp9 -crf 33 -b:v 0 \
  -vf "scale=1440:-2,fps=30" \
  -an \
  output.webm
```

- `-crf 33 -b:v 0` → quality mode for VP9. Range **31–36** (higher = smaller). `-b:v 0` is required so CRF actually controls quality.

> Skip AV1 for now. It's even smaller but very slow to encode and Safari support is patchy. H.264 (+ optional VP9) is the safe, fast combo.

### Make the poster image

Grab one frame as the poster (shown instantly before the video loads). Export it small and compressed:

```bash
ffmpeg -i output.mp4 -ss 00:00:01 -vframes 1 poster.jpg
```

Then compress `poster.jpg` (or convert to WebP) at [squoosh.app](https://squoosh.app) so it's under ~100 KB.

### Target sizes (sanity check)

| Video type | Length | Target size |
|---|---|---|
| Hero / background (muted, 1440p) | 8–15 s loop | **1–3 MB** |
| Section clip with sound (1080p) | 15–30 s | **3–6 MB** |
| Poster image | 1 frame | **< 100 KB** |

If your hero is still over ~4 MB: raise CRF (28 → 30), drop resolution (1440 → 1280), or shorten the loop.

---

## Step 2 — Upload to the CDN

Do **not** upload the video into Webflow Assets. Upload the encoded `.mp4`, optional `.webm`, and `poster.jpg` to our CDN (same jsDelivr/GitHub setup we already use for custom code — follow the dev-bible hosting rules, pin by tag/SHA, never `@main`).

Copy the final public URLs for each file. You'll paste them into the embed below.

---

## Step 3 — Embed in Webflow

Add an **Embed** element (not Background Video) where the video should go, and paste this:

### Hero / background video (autoplay, muted, loop)

```html
<video
  autoplay
  muted
  loop
  playsinline
  preload="metadata"
  poster="https://CDN-URL/poster.jpg"
  style="width:100%;height:100%;object-fit:cover;">
  <source src="https://CDN-URL/output.webm" type="video/webm">
  <source src="https://CDN-URL/output.mp4" type="video/mp4">
</video>
```

Rules that matter:
- **`muted` + `playsinline`** are required or autoplay is blocked on mobile (especially iOS).
- **`poster`** shows instantly so there's no blank gap while the video loads.
- **WebM `<source>` goes first**, MP4 second — the browser picks the first one it supports.
- **`preload="metadata"`** for the hero: loads just enough to start quickly without pulling the whole file up front.
- `object-fit:cover` makes it fill the container like a background.

### Below-the-fold videos → lazy load

For videos further down the page, don't load them until the user scrolls near them. Set `preload="none"` and put the URL in `data-src` instead of `src`, then load it on scroll.

Embed for each video:

```html
<video
  muted loop playsinline
  preload="none"
  poster="https://CDN-URL/poster.jpg"
  class="lazy-video"
  style="width:100%;height:100%;object-fit:cover;">
  <source data-src="https://CDN-URL/clip.mp4" type="video/mp4">
</video>
```

One script for the whole page (add once, in an Embed at the bottom of the page or in Page Settings → before `</body>`):

```html
<script>
document.addEventListener("DOMContentLoaded", function () {
  var videos = document.querySelectorAll("video.lazy-video");
  var io = new IntersectionObserver(function (entries, obs) {
    entries.forEach(function (entry) {
      if (!entry.isIntersecting) return;
      var video = entry.target;
      video.querySelectorAll("source[data-src]").forEach(function (s) {
        s.src = s.dataset.src;
      });
      video.load();
      video.play().catch(function () {});
      obs.unobserve(video);
    });
  }, { rootMargin: "200px" });
  videos.forEach(function (v) { io.observe(v); });
});
</script>
```

This loads each below-the-fold video only when it's about to enter the screen (`200px` before). The hero is not lazy-loaded — it should start immediately.

---

## Quick checklist per video

- [ ] Resolution set to 1280–1440px for hero (not blindly 1920)
- [ ] Frame rate 24 or 30 (not 60)
- [ ] Audio removed for muted/background videos (`-an`)
- [ ] `-movflags +faststart` used
- [ ] `-pix_fmt yuv420p` used (Safari/iOS safety)
- [ ] Final hero file is **1–3 MB**
- [ ] Poster image made and under ~100 KB
- [ ] Uploaded to CDN, not Webflow Assets
- [ ] `<video>` embed with `muted playsinline` + `poster`
- [ ] Below-the-fold videos lazy-loaded with the script
- [ ] Tested on iPhone Safari (autoplay + no black frame)

---

## Common mistakes

- **Using Webflow's Background Video element** → re-encodes badly, no size control. Always the manual `<video>` embed.
- **Forgetting `+faststart`** → video only plays after the whole file downloads. Page feels frozen.
- **Autoplay not working on mobile** → almost always a missing `muted` or `playsinline`.
- **No poster** → blank/black box while the video loads.
- **Loading all videos at once** → below-the-fold clips must be lazy-loaded, or the page pulls megabytes it doesn't need yet.
- **Encoding at 1920 + 60fps "just in case"** → this alone can 3–4x the file size for no visible benefit on a hero.
