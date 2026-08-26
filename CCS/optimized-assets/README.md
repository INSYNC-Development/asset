# CCS — optimized assets

Optimized per **Webflow Asset Optimization & Performance Audit — SOP** (Studio Vault →
`00 - Meta/Technical Guides/`) and its companion **Webflow Video Optimization** guide
(`webflow-video-optimization-guide.md`, Command A + Command C). Masters stay in
`CCS/Video/` (local archive, gitignored); only the files below are meant to be served.

## Video — "bordir machine" loop

| | Master | Optimized MP4 | Optimized WebM |
|---|---|---|---|
| File | `ccs_bordir_machine_15s_1920x1080.webm` | `videos/ccs_bordir_machine.mp4` | `videos/ccs_bordir_machine.webm` |
| Codec | VP9 + Opus | H.264 (CRF 28, preset slow) | VP9 (CRF 42, `-b:v 0`) |
| Size | 1920×1080 | 1440×810 | 1440×810 |
| FPS | 30 | 30 | 30 |
| Duration | 14.6 s | 14.6 s | 14.6 s |
| Audio | Opus | none (`-an`) | none (`-an`) |
| Weight | 12.8 MB | **2.05 MB** | **1.57 MB** |

WebM path **12.8 MB → 1.57 MB (−88%)**; MP4 fallback path **12.8 MB → 2.05 MB (−84%)**.
Most of the master's weight was the 1080p bitrate plus an Opus track that a muted
background loop never plays — both gone.

Both encodes sit inside the guide's 1–3 MB hero budget.

### Why CRF 28 (MP4) and CRF 42 (WebM)

This is live, high-detail footage — moving thread and needle plates — not the flat graphic
art in `deliver-releasing`, so the operating points differ from that folder's.

Measured against a lossless 1440p/30 reference (`x264 -qp 0`) built from the same master:

| Encode | Weight | SSIM |
|---|---|---|
| H.264 CRF 26 | 2.68 MB | 0.98708 |
| **H.264 CRF 28** | **2.05 MB** | **0.98460** |
| H.264 CRF 30 | 1.60 MB | 0.98148 |
| VP9 CRF 39 | 2.09 MB | 0.98726 |
| **VP9 CRF 42** | **1.57 MB** | **0.98548** |
| VP9 CRF 44 | 1.32 MB | 0.98419 |
| VP9 CRF 45 | 1.21 MB | 0.98346 |

H.264 stays at the guide's default CRF 28: CRF 26 costs +31% weight for +0.0025 SSIM, and
CRF 30 starts softening the thread detail that is the point of the shot.

VP9 is at **CRF 42, not the guide's 31–36 range**. That range is calibrated for flat
graphic art; on this footage CRF 33 lands at 3.87 MB — *heavier* than the MP4, which
defeats the reason for shipping a WebM at all. CRF 42 is the highest setting that still
beats the MP4's fidelity (0.98548 vs 0.98460) while coming in 23% lighter, which is the
20–40% saving the guide expects from VP9. CRF 44 is the first step that drops below the
MP4, so 42 is the boundary.

> **Measurement note.** Comparing a WebM against an MP4 reference with ffmpeg's `ssim`
> filter silently misaligns frames — Matroska carries a 1/1000 timebase, MP4 1/15360, and
> the filter warns `not matching timebases ... results may be incorrect`. Uncorrected, that
> pins every VP9 result near 0.964 regardless of CRF and makes VP9 look strictly worse than
> H.264. Normalize both inputs first:
>
> ```bash
> ffmpeg -i candidate.webm -i ref.mp4 \
>   -lavfi "[0:v]settb=AVTB,setpts=N/30/TB[a];[1:v]settb=AVTB,setpts=N/30/TB[b];[a][b]ssim" \
>   -f null -
> ```

### Commands used

```bash
# MP4 — guide Command A
ffmpeg -i ../Video/ccs_bordir_machine_15s_1920x1080.webm \
  -c:v libx264 -crf 28 -preset slow \
  -vf "scale=1440:-2,fps=30" -an \
  -movflags +faststart -pix_fmt yuv420p \
  videos/ccs_bordir_machine.mp4

# WebM — guide Command C, CRF retuned for live footage
ffmpeg -i ../Video/ccs_bordir_machine_15s_1920x1080.webm \
  -c:v libvpx-vp9 -crf 42 -b:v 0 -row-mt 1 -deadline good -cpu-used 2 \
  -vf "scale=1440:-2,fps=30" -an -pix_fmt yuv420p \
  videos/ccs_bordir_machine.webm

# Poster
ffmpeg -i videos/ccs_bordir_machine.mp4 -vframes 1 -q:v 3 \
  posters/ccs_bordir_machine_poster.jpg
```

Verified: `moov` before `mdat` (faststart ✓), `yuv420p` ✓, no audio track on either
encode ✓, 1440×810 ✓, 30 fps ✓, duration matches the master ✓.

## Poster

| File | Size | Weight |
|---|---|---|
| `posters/ccs_bordir_machine_poster.jpg` | 1440×810, sRGB | **76 KB** |

Under the guide's ~100 KB cap. Taken from frame 0 so the poster matches the first painted
frame of the loop — no jump when autoplay starts.

## Hosting

Serve from jsDelivr, **pinned by full commit SHA** — never `@main`, never unpinned:

```
https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/CCS/optimized-assets/videos/ccs_bordir_machine.mp4
https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/CCS/optimized-assets/videos/ccs_bordir_machine.webm
https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/CCS/optimized-assets/posters/ccs_bordir_machine_poster.jpg
```

## Embed (Webflow Embed element — not the Background Video element)

```html
<video
  autoplay muted loop playsinline
  preload="metadata"
  poster="https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/CCS/optimized-assets/posters/ccs_bordir_machine_poster.jpg"
  style="width:100%;height:100%;object-fit:cover;">
  <source src="https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/CCS/optimized-assets/videos/ccs_bordir_machine.webm" type="video/webm">
  <source src="https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/CCS/optimized-assets/videos/ccs_bordir_machine.mp4" type="video/mp4">
</video>
```

If the section sits below the fold, switch to `preload="none"` + `data-src` on the `<source>`
tags and let the page's IntersectionObserver lazy-loader start it (script in the
Webflow Video Optimization guide).
