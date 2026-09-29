# demo-viewport

A single-file debug page for inspecting a WebView from the inside. Open it in the
WebView you want to test, then read the numbers off the screen.

```
index.html
3p-cookie-test.html     # passive cross-site target for the cookie test
```

No build step, no dependencies, no network calls. Enable GitHub Pages for this
repo (Settings → Pages → deploy from the branch root) and load it inside the WebView
under test.

Neither filename starts with `_`: GitHub's branch deploy runs the site through
Jekyll, which silently drops any file whose name begins with an underscore.

## What it reports

| Section | What you get |
| --- | --- |
| *always visible* | Viewport size, `devicePixelRatio`, the four safe-area insets, `100dvh`, the `vh − dvh` gap, scroll position, and the active viewport meta |
| *always visible* | Toggles for each highlight, the unit test box picker, simulated keyboard height, and rotate / fullscreen / vibrate / file-picker buttons |
| viewport & units | `innerWidth/Height`, visual viewport, screen size, physical pixel size, `html` client size, `100%` behaviour inside a default-height body, scrollbar width, scroll position, and all eight units measured from real probe elements with a bar chart per axis |
| safe area | The four `env(safe-area-inset-*)` values, `env(max(...))`, whether `env()`/`constant()` parse, the active `viewport-fit`, the manual force override, and an inset simulator |
| keyboard | Live `innerHeight` vs `visualViewport.height`, largest shrink seen, `interactive-widget`, and an input to raise the real keyboard |
| pointer & hover | Pointer type, contact geometry, pressure, tilt and twist, `(hover: hover)`, `(pointer: coarse/fine/none)` and the `any-` variants, `maxTouchPoints`, tap latency, long-press timing, double taps, and a hover target that detects a stuck `:hover` |
| media queries | Colour scheme, `display-mode`, reduced motion, forced colours, gamut, plus a live grid of 36 queries that updates and logs every change |
| environment | User agent, UA client hints including high-entropy values, platform, languages, timezone, cores, memory, touch points, `isSecureContext`, `crossOriginIsolated`, referrer |
| storage & network | `localStorage`, `sessionStorage`, cookies, IndexedDB, CacheStorage, quota, a real third-party cookie test, `navigator.connection` |
| performance | FPS, long tasks, JS heap, navigation timing, transfer sizes, protocol |
| feature support | 55 CSS, DOM, platform and storage capability probes |
| event log | Timestamped viewport, visual viewport, media query, focus, pointer, key, clipboard and lifecycle events |
| viewport meta | `viewport-fit`, `interactive-widget` and `user-scalable` switches; each one reloads the page |

Everything below the first two blocks is collapsed by default, so the page opens
as a single screenful. Readouts update live while the page scrolls, so you can watch
the numbers change when the keyboard opens or a system bar animates.

## Safe area

The page applies the safe-area insets to its own chrome, so it behaves like a real
app instead of hiding its header under the status bar:

```css
:root  { --sat: env(safe-area-inset-top, 0px); /* …sar, --sab, --sal */ }
body   { padding: 0 var(--sar) calc(30vh + var(--sab)) var(--sal); }
.top   { padding: calc(8px + var(--sat)) calc(12px + var(--sar)) 8px calc(12px + var(--sal)); }
```

The top inset is padding *inside* the sticky header, so the header's background also
covers the status-bar strip instead of leaving a gap above it.

Two things to know:

- The insets are only non-zero with `viewport-fit=cover`. Without it the WebView
  insets the page itself and every value correctly reads `0px` — that is the answer,
  not a bug. Load `?fit=cover` to see the raw values.
- Some WebViews are edge-to-edge but still report `0px`. The **force insets** inputs
  in the safe area section set `--sat`/`--sar`/`--sab`/`--sal` by hand so you can pad
  the page as if the inset existed. They are stored in `localStorage`, because every
  viewport-meta change reloads the page. Clear a field to go back to `env()`.

## Overlays

- **Safe area** – tints the four inset regions, sized to the measured values, and
  labels each with its pixel size. Labels sit on the inner side of their band so they
  stay on screen even when the inset is `0px` and the band collapses to a line. One
  accent hue for all four: the label says which side it is.
- **Test box** – a fixed bar of `100vh` / `100dvh` / `100svh` / `100lvh` / `100%`
  pinned to the top, plus a `100dvw` bar pinned to the left, with dashed reference
  lines at the real viewport edges. If a bar of your app is cut off or leaves a gap,
  this reproduces it.
- **8px grid** and **device px grid** – one CSS pixel grid, and one grid line per
  device pixel so fractional scaling is visible.
- **Simulated keyboard** – a bottom sheet of a chosen height that does not resize
  the visual viewport, so you can check whether fixed bottom UI survives a real
  keyboard. Use the real keyboard for the actual `interactive-widget` behaviour.
- **Simulated insets** – type top/right/bottom/left to pad a box, for checking
  notch layouts on hardware that has none.

## Mobile rendering

The page is built for being read on the device it is debugging:

- A key/value row is a flex row with the label on the left and the value on the
  right. The value is `white-space: nowrap`, so it never breaks mid-number; only a
  long *label* can wrap. Values that genuinely cannot fit on one line — the user
  agent, paths, high-entropy hints — opt in to a stacked full-width row instead of
  being truncated.
- Both grid tracks used to be content-sized, which starved the value column and
  wrapped the user agent one character per line. Anything numeric is now
  `font-variant-numeric: tabular-nums` so digits line up between rows.
- `minmax(min(330px, 100%), 1fr)` for the card grid, so there is no horizontal
  overflow at 320px.
- On `(pointer: coarse)` controls grow to 40px and inputs go to 16px, so iOS does not
  zoom the page when you focus the keyboard test input.
- `-webkit-text-size-adjust: 100%` stops the system from inflating the readouts.
- Verified with no horizontal overflow and no wrapped values at 320, 375, 412, 768
  and 1280px wide, with every section both open and closed.

## Pointer, hover and tap behaviour

The media query section answers what the media queries claim; the pointer section
tells you what actually happened. That difference is the point, and it is where most
mobile bugs live:

- **Hover test box** — move the pointer over it. If it reports hovered while your last
  pointer was a touch, `:hover` is stuck after a tap, which is why a dropdown stays open
  on mobile.
- **`(hover: hover)` vs `(any-hover: hover)`** — a hybrid device with a mouse attached
  reports `hover: hover` even though nothing you can feel hovers. Compare the media query
  against the test box rather than trusting it.
- **`(pointer:)` vs `(any-pointer:)`** — `pointer` is the *primary* input. The primary
  pointer changes depending on the device, so a tablet with a mouse reports
  `(pointer: coarse)` while `(any-pointer: fine)` is also true. Styles keyed off the
  wrong one break on hybrid hardware.
- **Tap latency** — time from `pointerdown` to `click`. Above roughly 100ms means the
  300ms click delay or double-tap-zoom is still active and you need
  `touch-action: manipulation` or `user-scalable=no`.
- **Long press** — time from `pointerdown` to `contextmenu`. If it never fires, something
  is swallowing the gesture.
- **Double taps** — counted, with the resulting `visualViewport.scale` logged, so
  double-tap zoom is visible even when the page is not zoomed.
- **Live media query grid** — 36 queries including every `(hover:)`, `(pointer:)` and
  `(any-*)` variant, colour scheme, reduced motion/transparency/contrast, forced colours,
  `display-mode`, gamut, resolution and orientation. Every change is logged with a
  timestamp, so rotating the device or switching theme leaves a trace.

## URL parameters

Switching any of these reloads the page, so the settings are just query params:

| Param | Values | Effect |
| --- | --- | --- |
| `fit` | `cover`, `auto` | `viewport-fit` in the viewport meta |
| `iw` | `resizes-content`, `overlays-content` | `interactive-widget` in the viewport meta |
| `zoom` | `no` | `user-scalable=no, maximum-scale=1` |
| `scale` | number | `initial-scale` |

```
index.html?fit=cover&iw=resizes-content
```

### How the viewport meta is applied

The tag is static in the markup and the head script rewrites *that one tag* rather
than appending a new one:

```html
<meta name="viewport" id="viewport-meta" data-wvdbg content="width=device-width, initial-scale=1">
```

Two reasons, both of which matter more here than on a normal page:

- **No `width=980` window.** A JS-created meta does work — that is how the
  enable-pinch-zoom trick works in production — but the default viewport applies
  until the tag exists. A static tag removes the question of whether the script
  runs before first layout.
- **A WebView host can inject its own viewport meta.** With two tags, which one wins
  is implementation-defined, and the native side may also set `useWideViewPort` or
  `loadWithOverviewMode` behind your back. The script therefore adopts the first
  `meta[name=viewport]` it finds, tags it `id="viewport-meta"`, and rewrites it, so
  there is only ever one.

The page then reports what it found, which is the part worth knowing on a device:

- `own` — the static tag was there, normal case.
- `adopted` — a host had already injected one before the script ran. The host's
  values were replaced, which may not be what the host intended.
- `created` — no tag existed, so one was made.

If a host injects *after* load the tag count rises, and the line under the hero
readout says so: `2 viewport metas exist now, so a host is overriding one after
load`. That is usually the real cause of a viewport that will not behave.

## Getting data out

`copy` in the header writes a flattened text dump of every collected value plus the
event log to the clipboard. If the Clipboard API is unavailable (any non-secure
origin) a selectable textbox appears at the bottom of the page instead. The same
`collect()` object backs `download json` in the event log section, so the export
carries the full data set even though the page only displays a readable subset of it.
`download json` needs the WebView to allow downloads.

## Deploying

`Settings → Pages → Deploy from a branch`, branch `main`, folder `/ (root)`. No
build step and no workflow file. The site lands on
`https://amoshydra.github.io/demo-viewport/`.

Two things that bite people here:

- Files whose name starts with `_` are dropped by Jekyll, which is what branch deploy
  uses. Neither file in this repo does.
- GitHub Pages caches HTML for about ten minutes, so a freshly pushed change may not
  show up immediately. Append a cache-buster, which the page ignores and preserves
  when you change the viewport meta: `index.html?fit=cover&v=2`.
