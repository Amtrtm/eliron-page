# Elbit Interactive Landing Page Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-page Apple-clean, Elbit-branded landing page where a video-driven person's head follows the mouse via frame-accurate scrubbing, and six arch-arranged topics expand on hover.

**Architecture:** One self-contained `index.html` (inline CSS + JS). A re-encoded all-keyframe `<video>` is scrubbed by setting `currentTime` from a GSAP `quickTo`-smoothed "virtual frame" value driven by mouse Y; mouse X applies a faux-horizontal CSS transform. Six topic nodes are positioned along an SVG arch and reveal body copy via GSAP timelines on hover. Graceful degradation for touch/no-pointer and reduced-motion.

**Tech Stack:** HTML5 video, GSAP 3 (CDN), inline CSS/JS, SVG. Verification via the Playwright MCP browser tools and screenshots. ffmpeg for the asset re-encode.

**Verification note:** This is a visual/interaction static page, not a unit-testable module. "Tests" here are concrete browser-driven verification steps (navigate, evaluate JS state, screenshot, compare to expected). Each task ends with a verification step and a commit.

---

## File Structure

```
index.html                 ← the entire page (inline CSS + JS)
assets/person.mp4          ← re-encoded all-keyframe (GOP=1) clip
assets/elbit-logo.svg      ← recreated vector logo (navy + gold)
docs/superpowers/specs/2026-05-29-elbit-interactive-landing-design.md  ← spec (exists)
```

- `index.html` owns layout, styling, and all behavior. Single file by design (deliverable requirement). Behavior is organized into clearly-commented sections inside one `<script>`: config/content array, scrub engine, reveal engine, capability detection.
- `assets/` holds only static binary/vector assets.

---

## Task 1: Asset preparation (re-encode video + logo)

**Files:**
- Create: `assets/person.mp4`
- Create: `assets/elbit-logo.svg`

- [ ] **Step 1: Re-encode the source video to all-keyframe**

Run from the project root (PowerShell or bash):

```bash
mkdir -p assets
ffmpeg -y -i "grok-video-4b0fce80-b84b-47ed-8db1-8a05ec22b388.mp4" \
  -an -c:v libx264 -x264-params "keyint=1:scenecut=0" \
  -pix_fmt yuv420p -preset slow -crf 23 \
  -movflags +faststart assets/person.mp4
```

- [ ] **Step 2: Verify every frame is a keyframe and metadata is intact**

Run:

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=nb_frames,r_frame_rate,width,height -of default=noprint_wrappers=1 assets/person.mp4
# Count keyframes — should equal frame count (145)
ffprobe -v error -select_streams v:0 -show_frames -show_entries frame=key_frame -of csv assets/person.mp4 | grep -c "frame,1"
```

Expected: `nb_frames=145`, `r_frame_rate=24/1`, `752x416`, and keyframe count `145`.

- [ ] **Step 3: Create the Elbit logo SVG**

Recreate the Elbit Systems mark from the mockup as vector. Navy `#1B2A4A`, gold `#F2B705`. Create `assets/elbit-logo.svg`:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 240 70" role="img" aria-label="Elbit Systems">
  <!-- gold swoosh/mountain mark -->
  <path d="M150 44 L188 16 L210 30 L232 18 L232 44 Z" fill="#F2B705"/>
  <path d="M150 44 L188 16 L196 22 L160 44 Z" fill="#1B2A4A" opacity="0.15"/>
  <text x="0" y="40" font-family="Arial, Helvetica, sans-serif" font-weight="700"
        font-size="30" fill="#1B2A4A" letter-spacing="0.5">Elbit</text>
  <text x="0" y="60" font-family="Arial, Helvetica, sans-serif" font-weight="400"
        font-size="15" fill="#1B2A4A" letter-spacing="3">SYSTEMS</text>
</svg>
```

(Refine path coordinates against the mockup during build; this is a faithful starting vector.)

- [ ] **Step 4: Verify the SVG renders**

Open `assets/elbit-logo.svg` in a browser (or `mcp__plugin_playwright_playwright__browser_navigate` to the file URL) and confirm the wordmark + gold mark display with no parse errors.

- [ ] **Step 5: Commit**

```bash
git add assets/person.mp4 assets/elbit-logo.svg
git commit -m "chore: add re-encoded all-keyframe video and Elbit logo asset"
```

---

## Task 2: HTML scaffold + layout skeleton

Invoke the `frontend-design` skill before writing markup/CSS so the output meets the design-quality bar.

**Files:**
- Create: `index.html`

- [ ] **Step 1: Write the document shell, CSS variables, and stage**

Create `index.html`:

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Elbit Systems</title>
<style>
  :root{
    --navy:#1B2A4A; --gold:#F2B705; --ink:#1d1d1f;
    --muted:#6e6e73; --hair:#e5e5ea; --bg:#ffffff;
    --font:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Helvetica,Arial,sans-serif;
  }
  *{box-sizing:border-box;margin:0;padding:0}
  html,body{height:100%}
  body{background:var(--bg);color:var(--ink);font-family:var(--font);
    -webkit-font-smoothing:antialiased;overflow:hidden}
  .stage{position:relative;width:100vw;height:100vh;overflow:hidden}
  .logo-top{position:absolute;top:28px;left:50%;transform:translateX(-50%);width:170px;z-index:5}
  .person{position:absolute;left:-6vw;bottom:0;height:100vh;z-index:2;
    will-change:transform;pointer-events:none}
  .person video{height:100%;width:auto;object-fit:cover;display:block;
    /* organic soft mask echoing the mockup */
    -webkit-mask:radial-gradient(120% 90% at 60% 45%,#000 60%,transparent 78%);
            mask:radial-gradient(120% 90% at 60% 45%,#000 60%,transparent 78%)}
  .arch{position:absolute;inset:0;z-index:3;pointer-events:none}
  .arch svg{position:absolute;inset:0;width:100%;height:100%}
  .topics{position:absolute;inset:0;z-index:4}
</style>
</head>
<body>
  <main class="stage" id="stage">
    <img class="logo-top" src="assets/elbit-logo.svg" alt="Elbit Systems">
    <div class="person" id="person">
      <video id="vid" src="assets/person.mp4" muted playsinline preload="auto"></video>
    </div>
    <div class="arch"><svg id="archSvg" viewBox="0 0 1000 1000" preserveAspectRatio="none">
      <path id="archPath" fill="none" stroke="#d8dbe2" stroke-width="1.2"/>
    </svg></div>
    <div class="topics" id="topics"></div>
  </main>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"
          integrity="sha512-7Wt1Nn7B7+1F84Sb+nuJLwTpY/Cqp6+0PozPdEz1 vbnCAAAd-replace-with-verified-hash"
          crossorigin="anonymous" referrerpolicy="no-referrer"></script>
  <!-- NOTE: fetch the real SRI hash at build time:
       curl -s https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js | openssl dgst -sha384 -binary | openssl base64 -A
       then set integrity="sha384-<hash>". Do not ship the placeholder. -->

  <script>
    /* sections added in later tasks */
  </script>
</body>
</html>
```

- [ ] **Step 2: Verify layout renders with the person bleeding off the left edge**

Run (Playwright MCP): navigate to the local file URL, resize to 1440×900, take a screenshot.

```
mcp__plugin_playwright_playwright__browser_navigate  →  file:///<abs>/index.html
mcp__plugin_playwright_playwright__browser_resize     →  1440 x 900
mcp__plugin_playwright_playwright__browser_take_screenshot
```

Expected: white background, Elbit logo centered top, the person video tall on the left and clipped by the left edge. (GSAP loaded — verify via console has no errors.)

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: html scaffold, stage layout, person video bleeding off left edge"
```

---

## Task 3: Topic content model + arch placement

**Files:**
- Modify: `index.html` (the inline `<script>`)

- [ ] **Step 1: Add the content array and node-rendering code**

Inside the inline `<script>` (replace the placeholder comment):

```js
// ---- Content model (swap real copy/icons here later) ----
const TOPICS = [
  {id:'t1', title:'Topic One',   body:'Placeholder body copy for the first topic. Replace later.', icon:'mic'},
  {id:'t2', title:'Topic Two',   body:'Placeholder body copy for the second topic. Replace later.', icon:'battery'},
  {id:'t3', title:'Topic Three', body:'Placeholder body copy for the third topic. Replace later.', icon:'cursor'},
  {id:'t4', title:'Topic Four',  body:'Placeholder body copy for the fourth topic. Replace later.', icon:'gem'},
  {id:'t5', title:'Topic Five',  body:'Placeholder body copy for the fifth topic. Replace later.', icon:'signal'},
  {id:'t6', title:'Topic Six',   body:'Placeholder body copy for the sixth topic. Replace later.', icon:'mic'},
];

const ICONS = {
  mic:'<path d="M12 3a3 3 0 0 0-3 3v5a3 3 0 0 0 6 0V6a3 3 0 0 0-3-3Zm6 8a6 6 0 0 1-12 0M12 17v4" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round"/>',
  battery:'<rect x="3" y="8" width="16" height="8" rx="2" fill="none" stroke="currentColor" stroke-width="1.6"/><rect x="20" y="10" width="2" height="4" rx="1" fill="currentColor"/>',
  cursor:'<path d="M5 3l14 7-6 2-2 6z" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>',
  gem:'<path d="M6 4h12l3 5-9 11L3 9z" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>',
  signal:'<circle cx="12" cy="12" r="2.4" fill="currentColor"/><path d="M7 7a7 7 0 0 0 0 10M17 7a7 7 0 0 1 0 10" fill="none" stroke="currentColor" stroke-width="1.6"/>',
};

// ---- Arch geometry: concave curve, belly pointing left toward the head ----
// Returns {x,y} in viewBox(1000x1000) coords for fraction t in [0,1] top→bottom.
function archPoint(t){
  const topX=560, botX=560;          // endpoints near right
  const ctrlX=300, ctrlY=500;        // control pulled left = concave belly
  const topY=120,  botY=900;
  const x=(1-t)*(1-t)*topX + 2*(1-t)*t*ctrlX + t*t*botX;
  const y=(1-t)*(1-t)*topY + 2*(1-t)*t*ctrlY + t*t*botY;
  return {x,y};
}
```

- [ ] **Step 2: Draw the connector path and render the six nodes**

Append:

```js
// draw the arch path through the node points
(function drawArch(){
  const pts = TOPICS.map((_,i)=>archPoint(i/(TOPICS.length-1)));
  let d = `M ${pts[0].x} ${pts[0].y}`;
  for(let i=1;i<pts.length;i++){
    const p0=pts[i-1], p1=pts[i];
    const mx=(p0.x+p1.x)/2;
    d += ` Q ${mx} ${p0.y} ${p1.x} ${p1.y}`; // gentle curve between nodes
  }
  document.getElementById('archPath').setAttribute('d', d);
})();

const topicsEl = document.getElementById('topics');
TOPICS.forEach((t,i)=>{
  const p = archPoint(i/(TOPICS.length-1));
  const node = document.createElement('div');
  node.className='node';
  node.dataset.id=t.id;
  node.tabIndex=0;
  node.style.left = (p.x/1000*100)+'%';
  node.style.top  = (p.y/1000*100)+'%';
  node.innerHTML = `
    <div class="node-icon"><svg viewBox="0 0 24 24" width="22" height="22">${ICONS[t.icon]||''}</svg></div>
    <div class="node-card">
      <h3>${t.title}</h3>
      <p>${t.body}</p>
    </div>`;
  topicsEl.appendChild(node);
});
```

- [ ] **Step 3: Add node + card CSS (collapsed state)**

Add to `<style>`:

```css
.node{position:absolute;transform:translate(-50%,-50%);display:flex;align-items:center;
  gap:14px;cursor:pointer;pointer-events:auto}
.node-icon{flex:none;width:54px;height:54px;border-radius:50%;background:#fff;
  border:1px solid var(--hair);display:grid;place-items:center;color:var(--navy);
  box-shadow:0 6px 20px rgba(20,30,60,.06);transition:transform .35s,box-shadow .35s,border-color .35s}
.node-card{max-width:0;overflow:hidden;opacity:0;background:#fff;border-radius:16px;
  border:1px solid transparent;padding:0;will-change:max-width,opacity}
.node-card h3{font-size:17px;color:var(--navy);margin-bottom:6px;white-space:nowrap}
.node-card p{font-size:14px;line-height:1.5;color:var(--muted);max-width:300px}
.node:hover .node-icon,.node:focus-visible .node-icon{transform:scale(1.08);
  border-color:var(--gold);box-shadow:0 10px 28px rgba(20,30,60,.12)}
```

- [ ] **Step 4: Verify nodes sit on the arch top-to-bottom**

Navigate + screenshot (1440×900). Expected: six circular icons descending along a concave curve on the right, connector line threading through them, belly bowing left toward the person.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: topic content model, SVG arch geometry, six nodes on the arch"
```

---

## Task 4: Scrubbing engine (head follows mouse)

**Files:**
- Modify: `index.html` (inline `<script>`)

- [ ] **Step 1: Add the scrub engine**

Append to `<script>`:

```js
// ---- Scrub engine: mouse Y -> frame, mouse X -> faux-horizontal ----
const vid = document.getElementById('vid');
const personEl = document.getElementById('person');
const FPS = 24, DURATION_FALLBACK = 145/24;
let duration = DURATION_FALLBACK;
let lastSeek = -1;

vid.addEventListener('loadedmetadata', ()=>{ duration = vid.duration || DURATION_FALLBACK; });

// state object GSAP tweens; we read it each frame to drive the video
const state = { t:0.5, rx:0, tx:0 };

// quickTo gives us eased setters that we call every mousemove
const toT  = gsap.quickTo(state,'t',  {duration:0.5, ease:'power3'});
const toRx = gsap.quickTo(state,'rx', {duration:0.6, ease:'power3'});
const toTx = gsap.quickTo(state,'tx', {duration:0.6, ease:'power3'});

function onMove(e){
  const y = e.clientY / window.innerHeight;   // 0 top .. 1 bottom
  const x = e.clientX / window.innerWidth;    // 0 left .. 1 right
  toT(Math.min(1,Math.max(0,y)));             // top->frame0(up), bottom->last(down)
  toRx((x-0.5)*10);                           // faux yaw degrees
  toTx((x-0.5)*24);                           // subtle slide px
}

// render loop: apply eased state to the video
gsap.ticker.add(()=>{
  const target = state.t * duration;
  if(Math.abs(target-lastSeek) > 0.012){      // ~1/3 frame threshold, avoid stacking seeks
    vid.currentTime = target; lastSeek = target;
  }
  personEl.style.transform = `translateX(${state.tx}px) rotateY(${state.rx}deg)`;
});

// enable mouse tracking only when we have a fine pointer
const finePointer = window.matchMedia('(pointer:fine)').matches;
if(finePointer){
  vid.addEventListener('loadeddata', ()=>{ vid.currentTime = 0.5*duration; });
  window.addEventListener('mousemove', onMove, {passive:true});
  personEl.style.perspective='1200px';
}
```

- [ ] **Step 2: Verify scrubbing changes the frame with mouse Y**

Use Playwright to move the mouse and read `currentTime`:

```
mcp__plugin_playwright_playwright__browser_navigate → index.html
mcp__plugin_playwright_playwright__browser_evaluate →
  () => { window.dispatchEvent(new MouseEvent('mousemove',{clientX:1000,clientY:50}));
          return new Promise(r=>setTimeout(()=>r(document.getElementById('vid').currentTime),700)); }
```

Expected: returned `currentTime` is small (near 0 — head up). Repeat with `clientY: 850` → `currentTime` near `5.x` (head down). Screenshot both to confirm head pose differs.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: GSAP-smoothed video scrubbing — head follows mouse Y, faux yaw on X"
```

---

## Task 5: Hover reveal of topic body

**Files:**
- Modify: `index.html` (inline `<script>` + `<style>`)

- [ ] **Step 1: Add reveal logic (one open at a time)**

Append to `<script>`:

```js
// ---- Reveal engine ----
let openNode = null;
function openCard(node){
  if(openNode===node) return;
  if(openNode) closeCard(openNode);
  openNode = node;
  const card = node.querySelector('.node-card');
  card.style.borderColor='var(--hair)';
  gsap.to(card,{maxWidth:340, opacity:1, padding:'18px 22px',
    duration:0.45, ease:'power3.out'});
}
function closeCard(node){
  const card = node.querySelector('.node-card');
  gsap.to(card,{maxWidth:0, opacity:0, padding:0, duration:0.3, ease:'power2.in',
    onComplete:()=>{card.style.borderColor='transparent';}});
  if(openNode===node) openNode=null;
}

document.querySelectorAll('.node').forEach(node=>{
  node.addEventListener('mouseenter', ()=>openCard(node));
  node.addEventListener('mouseleave', ()=>closeCard(node));
  node.addEventListener('focus', ()=>openCard(node));
  node.addEventListener('blur',  ()=>closeCard(node));
});
```

- [ ] **Step 2: Add card shadow/elevation for the open state**

Add to `<style>`:

```css
.node-card{box-shadow:0 18px 50px rgba(20,30,60,.12)}
.node-card[style*="opacity: 0"]{box-shadow:none}
```

- [ ] **Step 3: Verify hover expands exactly one card**

```
mcp__plugin_playwright_playwright__browser_hover → first node (by ref from snapshot)
mcp__plugin_playwright_playwright__browser_take_screenshot
```

Expected: hovered node's card expands showing title + body; others stay collapsed. Hover a second node → first collapses, second opens.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: hover/focus reveal of topic body with GSAP, one open at a time"
```

---

## Task 6: Graceful degradation, reduced motion, accessibility

**Files:**
- Modify: `index.html` (inline `<script>` + `<style>`)

- [ ] **Step 1: Add touch/no-pointer fallback and reduced-motion handling**

Append to `<script>`:

```js
// ---- Capability fallbacks ----
const reduceMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if(!finePointer){
  // touch / no mouse: gentle auto-loop so the person feels alive; tap toggles cards
  vid.loop = true; vid.setAttribute('autoplay','');
  vid.play().catch(()=>{ vid.currentTime = 0.5*duration; });
  document.querySelectorAll('.node').forEach(node=>{
    node.addEventListener('click', ()=> openNode===node ? closeCard(node) : openCard(node));
  });
}

if(reduceMotion){
  // neutral pose, no scrub easing, instant reveals
  vid.removeEventListener('mousemove', onMove);
  vid.addEventListener('loadeddata', ()=>{ vid.currentTime = 0.5*duration; });
}
```

- [ ] **Step 2: Add reduced-motion CSS guard**

Add to `<style>`:

```css
@media (prefers-reduced-motion: reduce){
  .node-icon{transition:none}
}
```

- [ ] **Step 3: Verify touch + reduced-motion paths**

- Emulate no fine pointer: in a Playwright evaluate, confirm `window.matchMedia('(pointer:fine)').matches` branch — easiest check: load with touch context if available, or temporarily force `finePointer=false` and confirm tapping a node toggles its card (screenshot).
- Reduced motion: navigate, evaluate `matchMedia('(prefers-reduced-motion: reduce)').matches`; if true, confirm no console errors and head sits neutral.

Expected: page is interactive on all paths; no console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: touch tap-to-open fallback, reduced-motion support, a11y focus reveal"
```

---

## Task 7: Visual polish pass + final verification

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Polish against the mockup**

Compare a 1440×900 screenshot side-by-side with `Gemini_Generated_Image_jsq86rjsq86rjsq8.png`. Tune: arch curvature (`archPoint` control points), node spacing, logo size/position, the secondary Elbit wordmark near the person, person mask shape, gold accent usage. Add the secondary wordmark element if matching the mockup:

```html
<img class="logo-near" src="assets/elbit-logo.svg" alt="">
```
```css
.logo-near{position:absolute;left:14vw;top:42%;width:150px;z-index:3;opacity:.95}
```

- [ ] **Step 2: Full interaction smoke test**

Run through, capturing screenshots:
1. Load → head neutral, six nodes on arch, logo centered.
2. Mouse top-right → head up.
3. Mouse bottom-left → head down + slight slide.
4. Hover node 3 → card open, only one open.
5. No console errors at any step (`browser_console_messages`).

Expected: all pass, motion is smooth (no visible seek stutter thanks to GOP=1 + easing).

- [ ] **Step 3: Final commit**

```bash
git add index.html assets
git commit -m "polish: align layout with mockup, secondary wordmark, final verification"
```

---

## Self-Review (completed)

- **Spec coverage:** layout (T2,T3,T7), scrubbing approach A + faux-horizontal (T4), hover reveal (T5), Elbit/Apple palette & typography (T2,T7), content model swappable (T3), graceful degrade + reduced-motion + a11y (T6), re-encode all-keyframe asset (T1), single-file deliverable + GSAP CDN (T2). All spec sections map to a task.
- **Placeholder scan:** topic copy is intentionally placeholder per spec (swappable model in T3); no plan-level "TBD"/"handle edge cases" steps — every code step shows real code.
- **Type/name consistency:** `archPoint`, `TOPICS`, `ICONS`, `state{t,rx,tx}`, `toT/toRx/toTx`, `openCard/closeCard`, `openNode`, `finePointer`, `reduceMotion` used consistently across Tasks 3–6.

## Note on git

This folder is not currently a git repository. Run `git init` first (or tell me to) so the per-task commits work. Alternatively, skip commits and rely on the task checkpoints.
