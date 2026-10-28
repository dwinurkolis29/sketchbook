# Sketchbook Interactive Engine & Host Reference

This reference contains the exact structural, CSS, and mathematical building blocks extracted from the verified source ([`public/landing-pages/meng-to-sketchbook.html`](../../../../public/landing-pages/meng-to-sketchbook.html)). Use these patterns when building new sketchbook designs or hosting them in React.

---

## 1. Required DOM Skeleton for the 3D Stage & Loupe

The `.zoomwrap` and `.loupe` elements **must** live inside `.sb-3d` as siblings of `.sb-tilt` (outside the `rotateX`/`rotateY` transform) so the magnifying glass rests flat in screen space while the book tilts beneath it:

```html
<div class="sb-wrap" id="sbWrap">
  <!-- Horizontal motion blur filters for the intro riffle -->
  <svg width="0" height="0" style="position:absolute" aria-hidden="true">
    <filter id="sb-mblur-1"><feGaussianBlur stdDeviation="5 0"/></filter>
    <filter id="sb-mblur-2"><feGaussianBlur stdDeviation="14 0"/></filter>
  </svg>

  <div class="sb-stage" id="sbStage">
    <button class="sb-arrow left" id="sbLeft" aria-label="previous page">
      <svg viewBox="0 0 14 44" width="14" height="44" fill="none" aria-hidden="true">
        <polyline points="11,3 3,22 11,41" stroke="currentColor" stroke-width="1.1" stroke-linecap="round" stroke-linejoin="round"/>
      </svg>
    </button>

    <div class="sb-3d" id="sb3d">
      <div class="sb-tilt" id="sbTilt">
        <div class="sb-cast ambient" aria-hidden="true"></div>
        <div class="sb-cast contact" aria-hidden="true"></div>
        <div class="sb-cast hair" aria-hidden="true"></div>
        <div class="sb-book" id="sbBook"></div>
      </div>
      <!-- Cloned magnified book layer & draggable loupe -->
      <div class="zoomwrap" id="zoomWrap" aria-hidden="true">
        <div class="zoominner" id="zoomInner"></div>
      </div>
      <div class="loupe" id="loupe">
        <span class="grip"></span>
        <span class="ring">
          <span class="lens" id="loupeLens"><span class="mag" id="loupeMag"></span></span>
        </span>
      </div>
    </div>

    <button class="sb-arrow right" id="sbRight" aria-label="next page">
      <svg viewBox="0 0 14 44" width="14" height="44" fill="none" aria-hidden="true">
        <polyline points="3,3 11,22 3,41" stroke="currentColor" stroke-width="1.1" stroke-linecap="round" stroke-linejoin="round"/>
      </svg>
    </button>
  </div>

  <div class="sb-captions" id="sbCaptions"></div>

  <div class="sb-tools" role="group" aria-label="view controls">
    <button class="tool" id="zOut" aria-label="zoom out"><!-- minus magnifier SVG --></button>
    <span class="zoom-read" id="zRead">100%</span>
    <button class="tool" id="zIn" aria-label="zoom in"><!-- plus magnifier SVG --></button>
    <span class="tool-sep" aria-hidden="true"></span>
    <button class="tool" id="loupeBtn" aria-label="magnifier" aria-pressed="true"><!-- loupe SVG --></button>
  </div>

  <p class="sb-hint" id="sbHint">Drag the page to turn · Drag the glass across it</p>
</div>
```

---

## 2. 18-Strip 3D Page Curl Geometry & Lighting Math

Each turning leaf is constructed as a nested chain of `N = 18` `.strip` divs (`SPAN = 0.449`, `BETA = 0.60` radians). Because each strip is a child of the previous strip with `left: 100%` (or `right: 100%`) and `transform: rotateY(var(--td))`, the rotations accumulate into a smooth cylindrical arc.

```javascript
const N = 18;       // Number of nested strips
const SPAN = 0.449; // Width from spine gutter to outer page edge as fraction of book width
const BETA = 0.60;  // Peak arc curvature in radians

function buildCurl(dir, from, to) {
  strips = [];
  const c = el('div', 'curl ' + dir);
  c.style.setProperty('--n', N);
  c.style.setProperty('--span', SPAN);
  let host = c;
  for (let i = 0; i < N; i++) {
    const s = el('div', 'strip');
    s.style.setProperty('--i', i);
    const gut = 'calc(var(--bw) * 0.5)';
    const sw = 'calc(var(--bw) * ' + SPAN + ' / ' + N + ')';
    const A = 'calc(-1 * (' + gut + ' + ' + i + ' * ' + sw + '))'; // Front-facing offset
    const B = 'calc(' + (i + 1) + ' * ' + sw + ' - ' + gut + ')';  // Back-facing offset
    const f = el('div', 'face front'), b = el('div', 'face back');
    const dress = (e, url, px) => {
      e.style.backgroundImage = 'url(' + url + ')';
      e.style.backgroundPositionX = px;
    };
    dress(f, PAGES[from].url, dir === 'next' ? A : B);
    dress(b, PAGES[to].url,   dir === 'next' ? B : A);
    f.appendChild(el('div', 'sh')); f.appendChild(el('div', 'gl'));
    b.appendChild(el('div', 'sh')); b.appendChild(el('div', 'gl'));
    s.appendChild(f); s.appendChild(b);
    if (i === N - 1) s.classList.add('edge');
    host.appendChild(s); host = s;
    strips.push(s);
  }
  return c;
}

function applyTurn(t) {
  const th = Math.PI * t;
  const beta = BETA * Math.sin(Math.PI * t); // Zero curvature at t=0 and t=1, peak at t=0.5
  const D = 180 / Math.PI;
  const tt = th + beta, td = (2 * beta) / N;
  sb3d.style.setProperty('--tt', (tt * D).toFixed(2) + 'deg');
  sb3d.style.setProperty('--td', (td * D).toFixed(3) + 'deg');
  sb3d.style.setProperty('--shade', Math.sin(Math.PI * t).toFixed(3));
  fadeCaption(t);
  for (let i = 0; i < strips.length; i++) {
    const l1 = Math.abs(Math.cos(tt - i * td));
    const l2 = Math.abs(Math.cos(tt - (i + 1) * td));
    const st = strips[i].style;
    st.setProperty('--lit', l1.toFixed(3));
    st.setProperty('--a1', ((1 - l1) * 0.62).toFixed(3));
    st.setProperty('--a2', ((1 - l2) * 0.62).toFixed(3));
  }
}
```

---

## 3. Magnifying Glass (`.loupe`) & Cloned Zoom Layer Math

The loupe magnifies the page by `MAG = 2.3` by cloning `.sb-book`'s children (excluding `.sb-zone` hit buttons) into `#zoomInner`, applying a circular `radial-gradient` mask to `#zoomWrap`, and scaling `#zoomInner` about the exact page point `(px, py)` beneath the lens center `(lx, ly)`:

```javascript
const MAG = 2.3;

function syncZoomLayer() {
  zoomInner.textContent = '';
  for (const c of book.children) {
    if (c.classList.contains('sb-zone')) continue;
    zoomInner.appendChild(c.cloneNode(true));
  }
}

function placeLoupe() {
  if (lx === null) return;
  const B = { x: 0, y: 0, w: book.clientWidth, h: book.clientHeight };
  const bw = B.w, bh = B.h;
  if (!bw) return;
  const R = Math.round(Math.max(165, Math.min(262, bw * 0.235))) / 2;
  const bez = R * 2 * 0.058;
  loupe.style.setProperty('--lr', R * 2 + 'px');
  loupe.style.transform = `translate3d(${(lx - R).toFixed(1)}px, ${(ly - R).toFixed(1)}px, 0)`;
  if (loupeOn) loupe.classList.add('on');

  // Paper bounds inside the 1760x1240 image frame after zoom scale
  const z = view.z, cx = bw / 2, cy = bh / 2;
  const x0 = cx + (bw * 0.051 - cx) * z, x1 = cx + (bw * 0.949 - cx) * z;
  const y0 = cy + (bh * 0.218 - cy) * z, y1 = cy + (bh * 0.782 - cy) * z;

  const nx = Math.max(x0, Math.min(lx, x1));
  const ny = Math.max(y0, Math.min(ly, y1));
  const inside = (lx > x0 && lx < x1 && ly > y0 && ly < y1)
    ? Math.min(lx - x0, x1 - lx, ly - y0, y1 - ly)
    : -Math.hypot(lx - nx, ly - ny);
  const k = Math.max(0, Math.min(1, (inside + R * 0.30) / (R * 0.55)));

  zoomWrap.style.opacity = (loupeOn ? k : 0).toFixed(3);
  if (k <= 0.002) return;

  const r = (R - bez).toFixed(1);
  const mask = `radial-gradient(circle ${r}px at ${lx.toFixed(1)}px ${ly.toFixed(1)}px, #000 calc(100% - 1px), transparent 100%)`;
  zoomWrap.style.webkitMaskImage = mask;
  zoomWrap.style.maskImage = mask;

  const px = cx + (lx - cx) / z, py = cy + (ly - cy) / z, s = MAG * z;
  zoomInner.style.transform = `translate(${(lx - px * s).toFixed(1)}px, ${(ly - py * s).toFixed(1)}px) scale(${s.toFixed(4)})`;
}
```

---

## 4. React Host Wrapper & Customization Injection

To host the sketchbook HTML in a React project without `@designcodeio/threeui`, use a full-bleed sandboxed iframe and optionally inject CSS custom property overrides on load:

```tsx
import { useEffect, useRef, useState, type CSSProperties } from "react";

export interface SketchbookCustomization {
  headingFont?: "Instrument Serif" | "Newsreader" | "Geist";
  bodyFont?: "Newsreader" | "Geist" | "Instrument Serif";
  headingWeight?: 300 | 400 | 500 | 600;
  bodyWeight?: 200 | 300 | 400 | 500 | 600;
  primaryColor?: string; // Hex color for --ink (e.g. "#2b2721")
}

export interface SketchbookLandingPageProps extends SketchbookCustomization {
  sourceUrl?: string;
  title?: string;
  className?: string;
  style?: CSSProperties;
}

const SANDBOX_PERMISSIONS =
  "allow-downloads allow-forms allow-modals allow-popups allow-same-origin allow-scripts";

function hexToRgba(hex: string, alpha: number): string {
  const clean = hex.replace("#", "");
  const full = clean.length === 3 ? clean.replace(/./g, (c) => c + c) : clean;
  const r = Number.parseInt(full.slice(0, 2), 16);
  const g = Number.parseInt(full.slice(2, 4), 16);
  const b = Number.parseInt(full.slice(4, 6), 16);
  return `rgba(${r}, ${g}, ${b}, ${alpha})`;
}

function applySketchbookOverrides(
  iframe: HTMLIFrameElement | null,
  custom: SketchbookCustomization
) {
  const doc = iframe?.contentDocument;
  if (!doc?.head) return;

  const rules: string[] = [];
  if (custom.headingFont) {
    rules.push(`--display: '${custom.headingFont}', Georgia, serif;`);
  }
  if (custom.bodyFont) {
    rules.push(`--font: '${custom.bodyFont}', Georgia, serif;`);
  }
  if (custom.primaryColor && /^#([\da-f]{3}|[\da-f]{6})$/i.test(custom.primaryColor)) {
    rules.push(`--ink: ${custom.primaryColor};`);
    rules.push(`--ink-soft: ${hexToRgba(custom.primaryColor, 0.58)};`);
    rules.push(`--ink-faint: ${hexToRgba(custom.primaryColor, 0.36)};`);
    rules.push(`--hairline: ${hexToRgba(custom.primaryColor, 0.14)};`);
  }

  const extraCss: string[] = [];
  if (rules.length > 0) {
    extraCss.push(`:root { ${rules.join(" ")} }`);
  }
  if (custom.headingWeight) {
    extraCss.push(`.top .name, .plate .t { font-weight: ${custom.headingWeight}; }`);
  }
  if (custom.bodyWeight) {
    extraCss.push(`body, .bio { font-weight: ${custom.bodyWeight}; }`);
  }

  const styleId = "sketchbook-custom-overrides";
  const existing = doc.getElementById(styleId);
  if (extraCss.length === 0) {
    existing?.remove();
    return;
  }
  const styleEl = existing ?? doc.createElement("style");
  styleEl.id = styleId;
  styleEl.textContent = extraCss.join("\n");
  if (!existing) doc.head.appendChild(styleEl);
}

export function SketchbookLandingPage({
  sourceUrl = "/landing-pages/meng-to-sketchbook.html",
  title = "Meng To — Singapore Sketchbook",
  className = "",
  style,
  headingFont,
  bodyFont,
  headingWeight,
  bodyWeight,
  primaryColor,
}: SketchbookLandingPageProps) {
  const [ready, setReady] = useState(false);
  const iframeRef = useRef<HTMLIFrameElement | null>(null);

  useEffect(() => {
    applySketchbookOverrides(iframeRef.current, {
      headingFont,
      bodyFont,
      headingWeight,
      bodyWeight,
      primaryColor,
    });
  }, [headingFont, bodyFont, headingWeight, bodyWeight, primaryColor]);

  return (
    <div
      className={`landing-page-frame${className ? ` ${className}` : ""}`}
      data-state={ready ? "ready" : "loading"}
      style={{
        position: "relative",
        width: "100%",
        height: "100%",
        overflow: "hidden",
        background: "#ece7dc",
        ...style,
      }}
    >
      <iframe
        ref={iframeRef}
        title={title}
        src={sourceUrl}
        sandbox={SANDBOX_PERMISSIONS}
        loading="eager"
        onLoad={(e) => {
          applySketchbookOverrides(e.currentTarget, {
            headingFont,
            bodyFont,
            headingWeight,
            bodyWeight,
            primaryColor,
          });
          setReady(true);
        }}
        style={{
          position: "absolute",
          inset: 0,
          display: "block",
          width: "100%",
          height: "100%",
          border: 0,
          background: "#ece7dc",
        }}
      />
    </div>
  );
}
```

