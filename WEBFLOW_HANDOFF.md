# Excalibur Metals — 3D scroll maps: developer handoff

Three scroll-driven 3D maps for the Webflow site, built in plain Three.js (bundled; no other dependencies).

Each map is **one real 3D model**. As the visitor scrolls, it rotates from a top view into the Figma composition.
Then the claims, glows, gold blocks, pins, labels and project cards animate in on top of it.

| Page | Section on staging | `data-map` |
|---|---|---|
| `/` (home) | `section.section-map` "A District-Scale Gold Portfolio…" | `project` |
| `/north-tybo` | `section.section-map.is-flush` "Positioned in the Heart of an Active Mining District" | `northtybo` |
| `/bellehelen` | `section.section-map.is-flush` "Where it is" | `bellehelen` |

Each of these sections already contains an empty `div.map-embed-slot`. The map goes there.

A complete working example of all three maps with the same structure is in **`example/index.html`**. Serve the
package folder with any static server and open `example/`.

**v2 (2026-10-28): same maps, lighter on the browser.** The picture is unchanged (checked pixel by pixel against v1
at 13 points along each map's scroll). What changed:
- each marker's mask is fitted to its own shapes, instead of the whole frame plus a margin;
- the labels, pins, markers and cards are only updated when their progress changes, and the 3D model is only redrawn
  when its pose changes (or after a resize);
- the looping rings and ripples step 30 times a second instead of at the screen's refresh rate;
- on screens sharper than 1.5x, the 3D canvas renders at 1.5x.
Measured in Chrome on an AMD integrated GPU at 1920x945: with a map on screen and nobody scrolling, the graphics
process went from about 90 to 160 ms of work per second to about 25 to 35; while the 3D model turns, from about 100
to 200 to about 30 to 40. The assets did not change; `data-assets` may keep pointing at `@v1/assets/`.

---

## 1. What is in this package

```
excalibur-map.min.js        the script (~133 KB gzipped, Three.js included)
assets/                     project map (home)
assets/northtybo/           North Tybo
assets/bellehelen/          Bellehelen
example/index.html          working integration example (pinned scroll, links, card event)
WEBFLOW_HANDOFF.md          this file
```

Keep the folder structure as it is. The script finds each map's files at `<assets>/<map>/`.

## 2. Host the package

Webflow can't host `.js` files or folders, so upload the whole package to a static host that sends CORS headers.
WebGL needs CORS to read the images. Any of these works:

- **GitHub + jsDelivr:** push the folder to a repo and tag a release, e.g. `v1`. The base URL is then
  `https://cdn.jsdelivr.net/gh/<user>/<repo>@v1`.
- **Cloudflare Pages, Netlify or Vercel:** drag and drop the folder.
- **An S3 or R2 bucket:** add a CORS rule allowing `GET` from the site's domains (the staging `*.webflow.io` and the
  production domain).

Below, `BASE` is the URL of the folder that contains `excalibur-map.min.js`.

## 3. Add the script

Add it to each page that has a map (or once in Site Settings → Custom Code → Footer):

```html
<script src="BASE/excalibur-map.min.js" defer></script>
```

Custom code runs only in **Preview** and on the **published site**, not in the Designer canvas.

## 4. Put the map into the section

Inside each `.map-embed-slot`, add an **Embed** element with one of these.

**Home** (`/`):
```html
<div data-excalibur-map data-map="project" data-start-view="model" data-assets="BASE/assets/"
     data-link-bellehelen="/bellehelen" data-link-north-tybo="/north-tybo"></div>
```

**North Tybo** (`/north-tybo`):
```html
<div data-excalibur-map data-map="northtybo" data-start-view="model" data-assets="BASE/assets/"
     data-link-north-tybo=""></div>
```

**Bellehelen** (`/bellehelen`):
```html
<div data-excalibur-map data-map="bellehelen" data-start-view="model" data-assets="BASE/assets/"
     data-link-bellehelen=""></div>
```

- `data-assets` is the same `BASE/assets/` for all three maps. The map's own subfolder is added automatically.
- **Size:** the embed fills the slot's width and keeps the Figma frame's proportions, so its height follows its
  width. Size the slot as in the Figma layout: map on the left, copy column on the right; `.is-wide` on the
  project pages.
- **`data-start-view="model"`:** the map starts as the 3D model seen from straight above and rotates into place.
  On the home map, leaving it out gives the older start, a flat contour map that tilts up into relief.

## 5. Make the section scroll-driven (pinned)

The animation follows the scroll position through a **track**: the closest parent element that has the attribute
`data-xm-track`. The designed behaviour pins the map on screen while the visitor scrolls through the track.

In Webflow, on each map section:

1. **The track.** Select `section.section-map` and add the custom attribute `data-xm-track` (no value needed).
   Give the section a height of **300vh**: more for a slower animation, less for a faster one.
2. **The sticky part.** Select its child `div.map-layout` and set `Position: Sticky`, `Top: 0`, `Height: 100vh`.
   Centre its content vertically (Flex, align center) and set `Overflow: hidden`.
3. **Keep the parents unclipped.** Don't put `overflow: hidden` on the section or on any of its parents; that
   breaks `position: sticky`.

The equivalent CSS, also used in `example/index.html`:
```css
.section-map { height: 300vh; position: relative; }                       /* + attribute data-xm-track */
.map-layout  { position: sticky; top: 0; height: 100vh; display: flex; align-items: center; overflow: hidden; }
```

**Timing:** the animation progress runs from 0 at the moment the section's top reaches the top of the viewport, to
1 when its bottom reaches the bottom of the viewport. The model rotates during the first ~60%; the markers, pins
and cards come in during the rest.

**Without `data-xm-track`** the animation plays while the map crosses the viewport, with no pinning. That works,
but it is not the designed experience.

**Custom scroll setups** (e.g. GSAP ScrollTrigger) can drive a map directly:
```js
const map = document.querySelector('[data-map="bellehelen"]').__xm;
map.measure = () => {};          // stop the built-in scroll tracking
map.setProgress(0.5);            // 0 … 1
```

## 6. Cards are buttons: links and click handling

The **whole card** is clickable: the title, the text, the arrow and the empty area. It works on all three maps.

**Option A: plain links (attributes).** Put the destination on the embed:

| Attribute | Card |
|---|---|
| `data-link-bellehelen="/bellehelen"` | Bellehelen card (home, Bellehelen page) |
| `data-link-north-tybo="/north-tybo"` | North Tybo card (home, North Tybo page) |
| `data-link-target="_blank"` | optional: open in a new tab (`rel="noopener"` is added) |

A card with an empty link is not clickable and not focusable. With a link it is a real link: keyboard-focusable,
with an Enter key action and an accessible label.

**Option B: set links from JavaScript** (for example CMS-driven URLs):
```js
document.querySelectorAll('[data-excalibur-map]').forEach((el) => {
  el.__xm?.setLinks({ bellehelen: '/bellehelen', 'north-tybo': '/north-tybo', target: '' });
});
```
(`__xm` exists once the script has run. Run this after `DOMContentLoaded`, or call `window.ExcaliburMap.init()`
first.)

**Option C: your own action** (a modal, analytics, a route change). Every card click fires a bubbling event on the
embed:
```js
document.addEventListener('excalibur-card', (e) => {
  // e.detail.card: 'bellehelen' | 'north-tybo';  e.detail.href: the link ('' when none)
  openProjectModal(e.detail.card);
});
```
If the card has no `data-link-…`, nothing else happens, so the event is the only action.

A card becomes clickable when it has fully opened, at the end of its entrance animation.

## 7. Interactions already built in

- **Card hover:** the star turns one point and warms to gold, the arrow moves right and the border darkens.
  Everything returns smoothly on leave.
- **Label hover:** the navy pin head grows slightly.
- **Loops while visible:** gold glow ripples and navy radar rings run, and pause automatically off-screen.
- **Reduced motion:** with `prefers-reduced-motion`, the finished map is shown without scrubbing and with no hover
  motion.

## 8. Fonts

Labels and cards use the site's Webflow fonts, `"Aeonik Pro"` (500) and `"Aeonik Mono"` (400). If the font
families have different names in the Webflow project, add to the site's custom CSS:
```css
.xm-root { --xm-sans: "Aeonik Pro"; --xm-mono: "Aeonik Mono"; }
```

## 9. Performance and fallback

- **Lazy start:** nothing loads until a map is about half a screen from the viewport. A frame is rendered only
  when the scroll position changes.
- **Image weight:** about 0.4–0.7 MB per map on desktop. Screens up to about 2100 device pixels wide load the
  smaller versions. Device pixel ratio is capped at 2.
- **Fallback:** if WebGL is unavailable or an asset fails to load, the finished map is shown as an image, with the
  labels and the cards.

## 10. Troubleshooting

| Symptom | Fix |
|---|---|
| Nothing appears | The script URL or `data-assets` is wrong (check the Network tab for 404s), or you're in the Designer. Check in Preview or on the published site. |
| Map shows but stays at the start | `data-xm-track` is missing on the section, or the section isn't taller than the screen. |
| Map scrolls away instead of pinning | `position: sticky` is blocked by `overflow: hidden` on a parent, or `.map-layout` has no `top: 0` or `height: 100vh`. |
| Console error about CORS / tainted canvas | The host doesn't send `Access-Control-Allow-Origin`. Use jsDelivr, Pages or Netlify, or add a CORS rule. |
| Cards don't react | No `data-link-…` was set and no `excalibur-card` listener exists. Cards only become clickable once fully open. |
| Fonts look different | Set `--xm-sans` and `--xm-mono` (section 8). |

**Contact points in code:**
- `window.ExcaliburMap.init()` initialises any new `[data-excalibur-map]` elements, e.g. after a Webflow
  interaction injects one.
- `window.ExcaliburMap.instances()` lists the running maps.
