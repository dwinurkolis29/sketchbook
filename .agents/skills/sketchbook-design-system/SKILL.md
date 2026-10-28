---
name: sketchbook-design-system
description: >-
  Design and build tactile, editorial sketchbook landing pages and interactive portfolios
  inspired by the Meng To Singapore Sketchbook aesthetic. Use this skill when the user asks
  to create, adapt, or style a web page, portfolio, lookbook, or interactive showcase with
  a botanical paper atmosphere, 3D curled page turns, a draggable magnifying glass (loupe),
  pointer tilt physics, and editorial serif typography using Full HTML + DOM/CSS + JavaScript
  or a React host wrapper.
---

# Sketchbook Design System & Interactive Engine

Build tactile, physical-feeling editorial portfolios and illustrated lookbooks modeled on the verified Meng To Singapore Sketchbook implementation ([`public/landing-pages/meng-to-sketchbook.html`](../../../public/landing-pages/meng-to-sketchbook.html)).

This skill supports two workflows:
1. **Custom Design & Adaptation**: Creating new themed sketchbooks, portfolios, field journals, or product lookbooks using the tactile paper aesthetic, 3D page-curl engine, and draggable magnifier.
2. **Exact Verified Reproduction**: Hosting or porting the original Singapore Sketchbook byte-for-byte via an iframe or React wrapper without requiring `@designcodeio/threeui`.

---

## 1. Visual Direction & Design Tokens

The aesthetic sits at the intersection of **fine-press editorial typography** and **tactile physical craft** (watercolor paper wash, pressed botanicals, ink illustrations, and a brass-and-wood desk magnifier).

### Color Palette (CSS Custom Properties)
```css
:root {
  --paper: #ece7dc;                  /* Warm archival watercolor paper ground */
  --ink: #2b2721;                    /* Deep charcoal-sepia primary ink */
  --ink-soft: rgba(43, 39, 33, .58); /* Secondary copy, nav links, captions */
  --ink-faint: rgba(43, 39, 33, .36);/* Index numbers, metadata, hints, arrows */
  --hairline: rgba(43, 39, 33, .14); /* Subtle 1px editorial dividers & borders */
  --earth: #9a6a3e;                  /* Warm terracotta/ochre active accent */
}
```
- **Custom Theming**: When overriding `primaryColor` (ink), recalculate `--ink-soft` (`0.58` alpha), `--ink-faint` (`0.36` alpha), and `--hairline` (`0.14` alpha) from the new hex color while keeping `--earth` (`#9a6a3e`) or an explicit warm accent for active states.

### Typography System
- **Display / Headings (`--display`)**: `'Instrument Serif', "New York", Georgia, 'Times New Roman', serif`
  - Used for the top nameplate (`clamp(24px, 2.4vw, 30px)`), index plate titles (`clamp(19px, 2.1vw, 26px)`), and display callouts.
  - Weight: `400` (or customizable `300 | 400 | 500 | 600` when paired with `Newsreader` or `Geist`).
- **Body / Editorial Copy (`--font`)**: `'Newsreader', "New York", Georgia, 'Times New Roman', serif`
  - Used for long-form bio/editorial text (`clamp(17px, 1.7vw, 20px)`, `line-height: 1.74`, `font-weight: 300`, `max-width: 56ch`).
- **Micro-Caps & Labels**:
  - `--track-caps: .24em` with `text-transform: uppercase` for section labels (`11px`), hero kicker (`12px`), plate captions (`13px`), and footer (`11.5px`).
  - `--track-nav: .06em` for navigation links (`15px`, `font-weight: 300`).

---

## 2. Page Architecture & Layout Anatomy

Structure designs from top to bottom as a single, unhurried vertical scroll:

1. **Painted Wash Ground (`.wash`)**:
   - Fixed `inset: 0; z-index: -2` layer with a subtle watercolor texture (`bg-wash.jpg` at `opacity: .85`) overlaid with a vertical gradient fading into `--paper` (`#ece7dc`) at `88%`.
2. **Frosted Editorial Header (`.top`)**:
   - Fixed centered column (`name` above `nav`), backed by a masked `backdrop-filter: blur(14px) saturate(.95)` gradient (`mask-image: linear-gradient(180deg, #000 46%, transparent)`).
   - Navigation links separated from social/contact SVG icon buttons by a `1px solid var(--hairline)` vertical border.
3. **Interactive Sketchbook Hero Stage (`.hero`)**:
   - Full-viewport stage (`min-height: 100svh`), framed on desktop (`>900px`) by low-opacity botanical corner illustrations (`.botany.l`, `.botany.r` at `opacity: .5`).
   - Top uppercase kicker (`.hero-kicker`).
   - Centered 3D book stage (`.sb-stage`) with flanking prev/next chevron buttons (`.sb-arrow`), the 3D book container (`.sb-3d`, `max-width: 900px`, `aspect-ratio: 1760/1240`), caption crossfader (`.sb-captions`), pill toolbar (`.sb-tools`), and interaction hint (`.sb-hint`).
   - Breathing scroll-down chevron cue (`.hero-down`) at the bottom.
4. **Painted Watercolor Rules (`.rule` / `.rule.short`)**:
   - Horizontal brush-stroke divider image (`divider.png`) replacing harsh `<hr>` tags between major sections.
5. **Editorial About Section (`.about`)**:
   - Asymmetric 2-column grid (`grid-template-columns: minmax(0, 1fr) auto`, max-width `1080px`) pairing `.bio` prose on the left with a botanical illustration (`.bloom`) on the right.
6. **Numbered Plates Index (`.plates`)**:
   - Editorial table of contents (`.plate-list`) with top/bottom `--hairline` borders.
   - Each row (`.plate`) uses a 3-column baseline grid (`3.4em minmax(0, 1fr) auto`): zero-padded index number (`01`), `Instrument Serif` title, and uppercase location/category tag.
   - Hovering or active (`aria-current="true"`) shifts `padding-left` from `4px` to `12px`, adds a warm paper highlight (`rgba(255, 252, 244, .5)`), and colors the tag in `--earth`.
   - Clicking any row turns/jumps the sketchbook to that plate and smooth-scrolls back to `#sketchbook`.

---

## 3. Core Interactive Mechanics

Never replace these interactions with flat CSS fades or generic carousels. Refer to [`references/engine-reference.md`](./references/engine-reference.md) for the exact DOM structure, CSS rules, and JavaScript math:

1. **Curved 3D Leaf Page Turn (`N = 18` Nested Strips)**:
   - When stationary, render the spread as a single full image (`.sb-full`).
   - During a turn (`turn = { dir, from, to, t }`), split the book into static left/right halves (`.sb-half.left`, `.sb-half.right`) with dynamic gutter shadows (`.gutter-shade`), and mount a chain of **18 nested `.strip` elements** (`.curl.next` or `.curl.prev`).
   - Each strip bends incrementally by `td = (2 * beta) / N` where `beta = 0.60 * Math.sin(Math.PI * t)`, creating a realistic curved paper arc rather than a rigid flat door pivot.
   - Compute per-strip diffuse lighting (`--lit`, `--a1`, `--a2`) every frame from the cosine of each strip's world angle.
2. **Three-Layer Soft Desk Shadow (`.sb-cast`)**:
   - Three blurred radial gradients (`ambient`, `contact`, `hair`) sit beneath `.sb-book` and lighten dynamically (`--shade = Math.sin(Math.PI * t)`) while a page is lifted in mid-air.
3. **Draggable Magnifying Glass (`.loupe` + `.zoomwrap`)**:
   - Rendered in `.sb-3d`'s untransformed coordinate space so book tilt does not wobble the lens.
   - Consists of a wood/brass `.grip` handle, an annular metallic `.ring` bezel, a specular `.lens` dome, and a synchronized `.zoomwrap` layer containing a cloned DOM copy of the current book spread scaled by `MAG * view.z` (`MAG = 2.3`) and clipped via a circular `radial-gradient` `mask-image`.
   - **Edge Fade**: As the loupe center approaches or leaves the paper bounds, smoothly fade `.zoomwrap` opacity to `0` so the glass shows plain desk outside the book.
   - **Leaf Shove (`shoveLoupe`)**: Starting a page turn automatically nudges the loupe to the opposite lower corner (`0.12` or `0.88` width) if it sits in the path of the sweeping leaf.
   - Hidden automatically on coarse pointers (`@media (pointer: coarse)`).
4. **Restrained Pointer Tilt & View Spring**:
   - Moving a fine pointer tilts `.sb-tilt` toward the cursor up to `TILT_X = 4.5deg`, `TILT_Y = 7deg`, eased via a per-frame lerp (`e = 0.14`).
   - Toolbar zoom buttons scale `--zoom` between `0.9` (`90%`) and `1.5` (`150%`); double-clicking the stage resets zoom to `100%`.
5. **Intro Riffle with Motion Blur**:
   - On desktop initial load (when `prefers-reduced-motion` is false and pointer is fine), riffle through all spreads (`M + LAND` steps) along a sine bell velocity curve while applying horizontal SVG `<feGaussianBlur>` filters (`#sb-mblur-1`, `#sb-mblur-2`).

---

## 4. Implementation & Adaptation Workflow

### Mode A: Creating a New Design with the Sketchbook System
1. **Read the Engine Reference**: Load [`references/engine-reference.md`](./references/engine-reference.md) and inspect [`public/landing-pages/meng-to-sketchbook.html`](../../../public/landing-pages/meng-to-sketchbook.html) to ensure all CSS variables, 3D strip math, and loupe synchronization functions are preserved.
2. **Define the Plates Dataset (`PAGES`)**:
   - Prepare transparent-background open-sketchbook spread images (`1760×1240` aspect ratio recommended, where paper occupies roughly `x: 5.1%..94.9%`, `y: 21.8%..78.2%`).
   - Configure `PAGES = [{ file, title, place }, ...]` and set `LAND` to the index of the hero landing spread.
3. **Customize Tokens & Copy**:
   - Update `--paper`, `--ink`, `--earth`, `--display`, and `--font` in `:root`.
   - Update `.top .name`, `.hero-kicker`, `.about .bio`, and `.foot` editorial content.
4. **Verify Atmospheric & UI Assets**:
   - Ensure `bg-wash.jpg`, `divider.png`, `bloom.png`, `botany-left.png`, `botany-right.png`, and font files (`instrument-serif.woff2`, `instrument-serif-italic.woff2`, `newsreader.woff2`) or their custom replacements resolve at their relative paths.

### Mode B: Hosting the Verified Source in React (Zero-Dependency)
 When embedding the standalone HTML document inside a React application without `@designcodeio/threeui`:
1. Keep the complete HTML document at `/landing-pages/<name>.html` and its assets in `/landing-pages/<name>/`.
2. Mount it inside a sized, overflow-controlled container using a sandboxed `iframe` that retains `allow-downloads allow-forms allow-modals allow-popups allow-same-origin allow-scripts`.
3. If runtime typography or color customization props (`headingFont`, `bodyFont`, `headingWeight`, `bodyWeight`, `primaryColor`) are needed, inject an override `<style id="threeui-page-typography">` into `iframe.contentDocument.head` on load (see [`references/engine-reference.md`](./references/engine-reference.md)).

---

## 5. Verification & Guardrails

- **No Flat Approximations**: Never replace the 18-strip 3D page curl with a 2D slide, fade, or single flat `rotateY` card flip.
- **No Unreleased Loops**: Ensure the `requestAnimationFrame(tick)` loop self-terminates (`raf = null`) as soon as `spring`, `viewActive`, and `loupeEase()` settle.
- **Responsive & Accessibility Checks**:
  - Verify layout at desktop (`>900px`), tablet (`<=900px`, botany hidden), and mobile (`<=640px`, full-bleed stage, overlay arrows, loupe hidden on `pointer: coarse`).
  - Confirm `prefers-reduced-motion: reduce` skips the intro riffle, disables breathing animations, and commits page turns instantaneously.
  - Ensure all images decode before setting `document.body.dataset.ready = '1'` and triggering the intro riffle.

