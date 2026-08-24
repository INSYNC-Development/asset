# deliver-releasing — optimized assets

Optimized per **Webflow Asset Optimization & Performance Audit — SOP** (Studio Vault →
`00 - Meta/Technical Guides/`) and its companion **Webflow Video Optimization** guide.
Masters stay in `deliver-releasing/video/` (local archive); only the files below are
meant to be served.

## Video — homepage "about" graphic loop

| | Master | Optimized MP4 | Optimized WebM |
|---|---|---|---|
| File | `homepage_about_graphic_loop_24s_2264x2108.mp4` | `videos/homepage_about_graphic_loop.mp4` | `videos/homepage_about_graphic_loop.webm` |
| Codec | H.264 | H.264 (CRF 20, preset slow) | VP9 (CRF 30, `-b:v 0`) |
| Size | 2264×2108 | 1440×1340 | 1440×1340 |
| FPS | 60 | 30 | 30 |
| Duration | 24 s | 24 s | 24 s |
| Audio | none | none (`-an`) | none (`-an`) |
| Weight | 4.4 MB (+1.9 MB webm) | **432 KB** | **383 KB** |

Total payload **6.3 MB → 815 KB (−87%)**; MP4-only path **4.4 MB → 432 KB (−90%)**.

CRF 20 instead of the guide's default 28: the source is flat graphic art on black, so even
CRF 20 lands at 432 KB — far under the 1–3 MB hero budget — and the extra headroom buys
margin against banding in the dark gradients. Measured against a lossless 1440p/30 reference:
CRF 20 → SSIM 0.99973, CRF 24 → 0.99958, CRF 28 → 0.99931.

Commands used:

```bash
ffmpeg -i homepage_about_graphic_loop_24s_2264x2108.mp4 \
  -c:v libx264 -crf 20 -preset slow \
  -vf "scale=1440:-2,fps=30" -an \
  -movflags +faststart -pix_fmt yuv420p \
  videos/homepage_about_graphic_loop.mp4

ffmpeg -i homepage_about_graphic_loop_24s_2264x2108.mp4 \
  -c:v libvpx-vp9 -crf 30 -b:v 0 -row-mt 1 -deadline good -cpu-used 2 \
  -vf "scale=1440:-2,fps=30" -an -pix_fmt yuv420p \
  videos/homepage_about_graphic_loop.webm

ffmpeg -i videos/homepage_about_graphic_loop.mp4 -vframes 1 -q:v 3 \
  posters/homepage_about_graphic_loop_poster.jpg
```

Verified: `moov` before `mdat` (faststart ✓), `yuv420p` ✓, no audio track ✓, duration matches
the master ✓.

## Poster

| File | Size | Weight |
|---|---|---|
| `posters/homepage_about_graphic_loop_poster.jpg` | 1440×1340, sRGB | **22 KB** |

Taken from frame 0 so the poster matches the first painted frame of the loop — no jump when
autoplay starts.

## Hosting

Serve from jsDelivr, **pinned by full commit SHA** — never `@main`, never unpinned:

```
https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/deliver-releasing/optimized-assets/videos/homepage_about_graphic_loop.mp4
https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/deliver-releasing/optimized-assets/videos/homepage_about_graphic_loop.webm
https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/deliver-releasing/optimized-assets/posters/homepage_about_graphic_loop_poster.jpg
```

## Embed (Webflow Embed element — not the Background Video element)

```html
<video
  autoplay muted loop playsinline
  preload="metadata"
  poster="https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/deliver-releasing/optimized-assets/posters/homepage_about_graphic_loop_poster.jpg"
  style="width:100%;height:100%;object-fit:cover;">
  <source src="https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/deliver-releasing/optimized-assets/videos/homepage_about_graphic_loop.webm" type="video/webm">
  <source src="https://cdn.jsdelivr.net/gh/INSYNC-Development/asset@<full-sha>/deliver-releasing/optimized-assets/videos/homepage_about_graphic_loop.mp4" type="video/mp4">
</video>
```

If the section sits below the fold, switch to `preload="none"` + `data-src` on the `<source>`
tags and let the page's IntersectionObserver lazy-loader start it (script in the
Webflow Video Optimization guide).
