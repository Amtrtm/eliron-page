# Elbit Systems — Interactive Landing Page

[![Live Demo](https://img.shields.io/badge/live%20demo-amtrtm.github.io%2Feliron--page-2B9348.svg)](https://amtrtm.github.io/eliron-page/)
[![Tech](https://img.shields.io/badge/tech-HTML5-E34F26.svg)](https://developer.mozilla.org/docs/Web/HTML)
[![Animation](https://img.shields.io/badge/GSAP-3.12.5-88CE02.svg)](https://gsap.com/)
[![Type](https://img.shields.io/badge/project-Landing%20Page-1B2A4A.svg)](#)
[![Platform](https://img.shields.io/badge/platform-Web-blue.svg)](#)

**▶ Live demo:** https://amtrtm.github.io/eliron-page/ — move your mouse over the page; the person's head follows the cursor and the topics reveal on hover.

A single-file, self-contained interactive landing page. A person video reacts to the
cursor — moving the mouse up/down scrubs the video frame (head looks up/down) and moving
left/right adds a subtle parallax slide and yaw. Six topic nodes sit along a concave arch;
hovering (or focusing) a node reveals its card, with only one card open at a time. The
design aims for an Apple-clean aesthetic.

The whole experience lives in `index.html` (markup, styles, and script inline). The only
external dependency is GSAP, loaded from a CDN.

## Features

- Cursor-driven video scrubbing (mouse Y → frame, mouse X → parallax).
- Six topic nodes positioned on an SVG arch with reveal-on-hover cards.
- Graceful degradation:
  - **Touch / no fine pointer:** the video gently auto-loops so the person feels alive,
    and tapping a node toggles its card.
  - **`prefers-reduced-motion`:** scrub easing is disabled, the head holds a neutral pose,
    and reveals are instant.
- Keyboard accessible nodes (focus/blur open and close cards).

## Preview locally

The video is **scrubbed by setting `currentTime`**, which requires the server to honor
HTTP **Range** requests (responding with `206 Partial Content`). Without Range support,
seeking will stall or fail and the head will not react to the cursor.

> ⚠️ Python's stock `http.server` does **NOT** support Range requests — video scrubbing
> will fail under it. Use a Range-capable static server instead.

Recommended:

```bash
npx serve .
```

Then open the printed URL (e.g. `http://localhost:3000`). Any static server that returns
`206 Partial Content` for ranged requests works.

To confirm Range support:

```bash
curl -s -D - -o /dev/null -H "Range: bytes=0-99" http://localhost:3000/assets/person.mp4
# Expect: HTTP/1.1 206 Partial Content  +  Accept-Ranges: bytes
```

## Deployment requirement

Whatever host you deploy to **must serve `assets/person.mp4` with Range / `206` support.**
Most CDNs and modern static hosts (Netlify, Vercel, GitHub Pages, S3/CloudFront, nginx)
do this by default. If the head stops reacting to the cursor in production, the host is
almost certainly not honoring Range requests for the video.

## File layout

```
index.html              # the entire page (HTML + CSS + JS inline)
assets/person.mp4       # the person video (scrubbed by the cursor)
assets/elbit-logo.png   # Elbit Systems logo (top-center)
README.md
```

## Swapping the 6 topics

All topic content and icons live in `index.html`, near the top of the inline `<script>`:

- **`TOPICS`** — an array of six objects: `{ id, title, body, icon }`. Edit `title` and
  `body` for each topic's copy. The `icon` value must be a key in the `ICONS` map.
- **`ICONS`** — a map of icon name → inline SVG path markup (drawn inside a `0 0 24 24`
  viewBox, using `currentColor`). Add a new entry here to introduce a new icon, then
  reference its key from a topic's `icon` field.

The arch curve and node placement are derived automatically from the number of topics via
the `archPoint(t)` function and the node-render loop, so the six nodes stay on the curve.

> **For collaborators:** the quickest way to edit copy is to open `index.html` on GitHub,
> click the pencil (✎) to edit in the browser, change the `title`/`body` strings in the
> `TOPICS` array, and commit. GitHub Pages rebuilds the live demo automatically within a
> minute or two.
