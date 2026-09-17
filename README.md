# wic-meditation

Static site hosting guided practices for Wisdom Integration Coaching.

**Live:** https://wisdomintegrationcoaching.josephbumbulis.com
**Media:** https://media.josephbumbulis.com (Cloudflare R2, bucket `meditation`)

---

## What's here

| Path | URL | What it is |
|---|---|---|
| `index.html` | `/` | Guided meditation — audio player with breathing dial |
| `breathwork/index.html` | `/breathwork` | Breathwork video, embedded from YouTube |
| `404.html` | any unmatched path | Branded error page |

Three HTML files. No build step, no dependencies, no framework. Each page is
fully self-contained — styles and scripts are inline.

---

## Architecture

The two practices are hosted differently, on purpose.

```
GitHub repo  ──push──▶  Cloudflare Pages  ──▶  wisdomintegrationcoaching.josephbumbulis.com
   (HTML only)                                    │
                                    audio ────────┤
                                                  ▼
                            Cloudflare R2 (bucket: meditation)
                                    ──▶  media.josephbumbulis.com
                                                  │
                                    video ────────┤
                                                  ▼
                                              YouTube
```

**Audio is self-hosted in R2** because it's small, it's yours, and there's no
reason to send someone to a third party for a ten-minute meditation.

**Video is on YouTube** because video is large, YouTube handles adaptive
bitrate for free, and the video already exists there. The tradeoff is that
YouTube's player takes over once someone presses play.

**Nothing media-related lives in this repo.** Cloudflare Pages caps a single
asset at 25 MiB, and git handles large binaries badly — every version of a
big file stays in history forever. Media extensions are gitignored.

---

## Deploying

Push to `main`. That's the whole process.

```bash
git add -A
git commit -m "Update practice notes"
git push
```

Cloudflare Pages watches the repo and rebuilds on every push to `main`,
usually in under a minute. Pushes to any other branch produce a preview
deployment at its own URL — useful for trying changes without touching the
live site.

The free plan allows 500 builds per month.

---

## The audio file (R2)

Bucket `meditation` · served at `https://media.josephbumbulis.com/Internal%20scan%20meditation.m4a`

| File | Used by |
|---|---|
| `meditation.m4a` | `/` |

Note that the bucket's name and the domain it's served from are unrelated —
the bucket happens to be called `meditation`, but every URL is built from
the custom domain. Renaming the bucket would not change any URL.

### Replacing the audio

Upload the new version to R2 under the **same filename** and it goes live
immediately — no deploy needed, nothing in this repo changes.

Cloudflare caches aggressively on a custom domain, so an updated file under
the same name may keep serving the old version for a while. Two options:
purge the cache from the Cloudflare dashboard, or upload under a new
filename (`meditation-v2.m4a`) and update the constant below.

### Changing where the audio lives

Near the top of the `<script>` in `index.html`:

```js
/* ══════════════════════════════════════════════════
   MEDIA SOURCE — the only line you change if the
   file moves.
   ══════════════════════════════════════════════════ */
var AUDIO_SRC = 'https://media.josephbumbulis.com/meditation.m4a';
```

Change that URL and nothing else. A relative path like `'meditation.m4a'`
also works if you ever put a small file beside the HTML.

### Format

`.m4a` (AAC) or `.mp3` — both play natively in every current browser.

---

## The video (YouTube)

Near the top of the `<script>` in `breathwork/index.html`:

```js
var VIDEO_ID = 'REPLACE_WITH_VIDEO_ID';
var RUNTIME  = '22 minutes';
```

`VIDEO_ID` is the part after `v=` in a YouTube URL, or after the slash on a
`youtu.be` link:

```
youtube.com/watch?v=dQw4w9WgXcQ  →  dQw4w9WgXcQ
youtu.be/dQw4w9WgXcQ             →  dQw4w9WgXcQ
```

`RUNTIME` is display text only — nothing reads it from YouTube. Set it to
`''` to hide the label.

Swapping in a different video means changing one line and pushing.

### How the embed works

The page does **not** load a YouTube iframe on page load. Instead it shows
YouTube's own thumbnail behind the branded veil, and only injects the iframe
when someone presses play. Two reasons: the page stays fast, and nobody who
doesn't watch the video gets YouTube cookies or tracking.

The iframe uses `youtube-nocookie.com` (YouTube's privacy-enhanced domain)
with `rel=0`. Be aware that `rel=0` no longer removes end-of-video suggestions
entirely — it limits them to your own channel. If suppressing them completely
matters, self-hosting is the only way, and that means R2 or Cloudflare Stream
instead.

The thumbnail tries `maxresdefault.jpg` first and falls back to
`hqdefault.jpg`, since not every video has a max-resolution still.

---

## Working locally

Open `index.html` directly and most of it works, but `file://` URLs behave
oddly with media and absolute paths. Better to serve it:

```bash
python3 -m http.server 8000
```

Then visit `http://localhost:8000`. The audio still streams from R2 and the
video still comes from YouTube, so you're testing the real thing.

---

## Adding a page

1. Create `newpage/index.html` — copy an existing page so the design system
   comes along.
2. Add a link in the `<footer>` of the other pages.
3. Commit and push.

The folder name becomes the URL: `newpage/index.html` serves at `/newpage`.

---

## Design system

From the WIC.KED brand book. Every page uses these and nothing else.

```css
--green:      #2D4A3E;   /* Forest Green */
--green-deep: #1A2E28;   /* Deep Forest — page background */
--sand:       #E8DCC8;   /* body text on dark */
--cream:      #F5F0E8;   /* light backgrounds */
--pink:       #FF3D8A;   /* accent only */
```

**Type** — Cormorant Garamond (300 / 400 / 600) for everything editorial;
Barlow (300–600) for kickers, labels, and UI. Loaded from Google Fonts.

**Structural vocabulary** — 5px pink bar down the left edge, pink gradient
bar along the bottom, SVG grain overlay at ~45% opacity, corner triangle at
6% opacity, 48px pink hairline rule under headlines.

**Rules worth keeping** — pink is an accent and never a background. Bold only
for genuine emphasis. Let the copy breathe.

---

## Known gotchas

- **Don't enable the R2 `r2.dev` development URL.** It's rate-limited and
  uncached. The custom domain is the one to use.
- **The apex domain is not part of this.** `josephbumbulis.com` serves a Kit
  landing page and its DNS record must stay unproxied (grey cloud) for Kit's
  certificate to work. This subdomain is independent of that.
- **Safety copy on `/breathwork` is yours to own.** The contraindications
  listed there are a generic first draft and should be replaced with whatever
  you actually screen for.
- **Cross-origin audio works without CORS headers** for plain `<audio>`
  playback. If you later add anything that reads the audio programmatically
  (a waveform, the Web Audio API), you'll need to configure CORS on the
  bucket.
