# Elbit Systems — Interactive Scrubbing Landing Page

**Date:** 2026-05-29
**Status:** Approved design, pending implementation plan

## Overview

A single-page, creative landing page with an Apple-clean white aesthetic and Elbit
Systems branding. A person (video) sits on the left, partially cut off by the viewport
edge. Six topic nodes are arranged along a concave arch on the right, running top to
bottom, with the belly of the arch pointing left toward the person's head. As the user
moves the mouse, the person's head follows via frame-accurate video scrubbing. Hovering a
topic expands a card revealing that topic's body content.

Reference mockup: `Gemini_Generated_Image_jsq86rjsq86rjsq8.png` (text/icons are
placeholders — to be replaced later).

## Goals

- Distinctive, polished, production-grade feel (no generic AI aesthetic).
- Head tracks the mouse smoothly with no seek jank.
- Topics reveal body content on hover.
- Always works: graceful degradation on touch/no-pointer devices.

## Non-Goals

- Final marketing copy or final icons (placeholders now, swappable later).
- A build pipeline / framework. Plain HTML + GSAP via CDN.
- Backend, analytics, or CMS integration.

## Assets

- **Source video:** `grok-video-4b0fce80-b84b-47ed-8db1-8a05ec22b388.mp4`
  - 752×416, 24 fps, 145 frames, ~6.04 s, H.264.
  - Content: a single continuous **vertical head tilt** — frame 0 looking up, middle
    looking forward, final frame looking down. No left↔right rotation.
- **Re-encoded video:** `assets/person.mp4` — same clip re-encoded with **GOP=1
  (every frame a keyframe / all-intra)** so any `currentTime` seek is instant. This is
  the asset the page loads.
- **Logo:** `assets/elbit-logo.svg` — recreated as vector from the mockup (navy + gold).

## Layout

Full-viewport white stage.

- **Person video:** anchored bottom-left, oversized, bleeding off the left edge
  (partially cut). Masked in a soft organic/rounded shape echoing the mockup.
- **Primary Elbit logo:** top-center.
- **Secondary Elbit wordmark:** near the person, as in the mockup.
- **Six topic nodes:** right side, positioned along a concave SVG arch from top to
  bottom; arch belly points left toward the head. Each node = circular icon + short
  title + hidden body copy.
- **Connector line:** faint SVG path threading through all six nodes.

## Interaction

### Head tracking (scrubbing)
- **Approach A (approved):** one HTML5 `<video>` element; a tweened "virtual frame"
  value drives `video.currentTime`.
- **Mouse Y → vertical tilt:** cursor near top of stage maps to frame 0 (head up),
  cursor near bottom maps to final frame (head down). This reinforces the top→bottom
  topic arch.
- **Mouse X → faux horizontal:** a subtle CSS `rotateY`/`translateX` on the video
  element, proportional to cursor X offset from center, to fake a slight left-right turn.
- **Smoothing:** GSAP `quickTo` eases the virtual-frame value and the X-transform toward
  their targets so the head glides rather than snaps. Mouse handler is rAF-throttled.
- The re-encoded GOP=1 video makes per-frame seeks instant; combined with easing this
  yields smooth motion.

### Topic reveal
- **Hover to open** (approved): hovering a node runs a GSAP timeline that expands a
  rounded card showing the body copy; leaving collapses it. Only one open at a time.
- Hovered node gets a subtle scale/parallax lift. Arch has a soft idle float.

## Color & Typography (Elbit + Apple-clean)

- Background: pure white, generous whitespace, thin hairlines.
- Font: SF-like system stack (`-apple-system, BlinkMacSystemFont, "Segoe UI", ...`).
- Palette: Elbit corporate **navy/blue (~`#1B2A4A`)** + signature **yellow-gold**
  accent; neutral grays for body text. Accents reserved for active/hover states.

## Content Model

Six placeholder topics defined in a single JS array (or `data-` attributes) so real
content and icons swap in trivially later. Each item:

```js
{ id, title, body, icon }  // icon = inline SVG or symbol id
```

## Graceful Degradation & Accessibility

- **No fine pointer / touch:** head plays a gentle auto-loop or rests in a neutral pose;
  topics become **tap-to-open** instead of hover.
- **`prefers-reduced-motion`:** disable idle float and scrubbing easing; head sits in a
  neutral pose; reveals become instant fades.
- Topics keyboard-focusable; reveal also triggers on focus.
- Video has no audio dependency; muted, `playsinline`, `preload="auto"`.

## Technical Notes

- **Deliverable:** single self-contained `index.html` (inline CSS + JS), GSAP from CDN
  (core; `quickTo` is part of core gsap). Video referenced locally from `assets/`.
- **Seek robustness:** wait for `loadeddata`; guard seeks so we don't stack requests
  (only set `currentTime` when the value meaningfully changed).
- Re-encode command (reference): all-intra H.264, e.g.
  `ffmpeg -i src.mp4 -an -c:v libx264 -x264-params keyint=1 -preset slow -crf 20 person.mp4`.

## Deliverable Structure

```
index.html              ← the page (inline CSS/JS)
assets/person.mp4       ← re-encoded all-keyframe clip
assets/elbit-logo.svg   ← recreated vector logo
```

## Open Questions / Future

- Real topic copy + final icons (later pass).
- Optional: pin-on-click in addition to hover (deferred).
