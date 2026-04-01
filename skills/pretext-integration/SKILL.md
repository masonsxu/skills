---
name: pretext-integration
description: |
  How to integrate and use @chenglou/pretext — a pure JS/TS library for fast, DOM-free multiline text measurement and layout. Trigger this skill whenever the user mentions: text measurement, text height calculation, text layout, line breaking, word wrapping, canvas text rendering, virtualized text, shrinkwrap text, custom text layout, DOM-free text measurement, text overflow detection, layout shift prevention, chat bubble sizing, message bubble width, textarea height prediction, virtual scroll height, PDF text layout, SVG text layout, WebGL text rendering, text-around-obstacles layout, multi-column text flow, streaming text layout, or anything related to measuring or laying out text in the browser without DOM reflow. Also trigger on mentions of pretext, @chenglou/pretext, avoiding getBoundingClientRect / offsetHeight for text sizing, or replacing DOM-based text measurement. Covers both simple height measurement and advanced manual layout (canvas rendering, SVG text, text-around-obstacles, shrinkwrap, streaming line-by-line layout). Also relevant for React text virtualization, chat UI sizing, and any scenario where text dimensions must be known before rendering.
---

# Pretext Integration Guide

Pretext (`@chenglou/pretext`) measures and lays out multiline text without touching the DOM. It uses canvas `measureText` under the hood and caches segment widths so repeated layout calculations (e.g., on resize) are pure arithmetic — no layout reflow, no DOM reads.

**Input**: `$ARGUMENTS` — optional; the specific pretext use case or feature to implement.

## Core Concept: Two-Phase Design

Everything in Pretext splits into **expensive preparation** and **cheap layout**:

1. **`prepare()` / `prepareWithSegments()`** — Runs once when text first appears. Segments the text, measures each segment via canvas, caches widths. ~19ms for 500 texts.
2. **`layout()` / `layoutWithLines()`** — Runs on every resize or width change. Pure arithmetic over cached widths. ~0.09ms for 500 texts.

**Do not re-prepare when only the width changes.** Prepare once, layout many times. This is the fundamental performance contract — violating it negates the library's purpose.

## Installation

```sh
npm install @chenglou/pretext
```

## Use Case 1: Measure Paragraph Height

Know how tall a block of text will be at a given width — for virtualization, layout shift prevention, overflow detection:

```ts
import { prepare, layout } from '@chenglou/pretext'

const prepared = prepare('Hello world', '16px Inter')
const { height, lineCount } = layout(prepared, 200, 20)
```

### Font and lineHeight parameters

The `font` string must match what you'd set on `ctx.font` — standard CSS font shorthand:

```ts
'16px Inter'
'bold 14px "Helvetica Neue", Helvetica, sans-serif'
'italic 20px Georgia, serif'
```

**This string must match your CSS declaration exactly.** Mismatched fonts mean mismatched measurements. The `lineHeight` parameter must also match your CSS `line-height`.

**Avoid `system-ui` on macOS** — canvas and DOM resolve different font variants, causing measurement drift. Use named fonts.

### When to re-prepare vs re-layout

```ts
const prepared = prepare(text, font)

function onResize(newWidth: number) {
  const { height, lineCount } = layout(prepared, newWidth, lineHeight)
}
```

Re-prepare only when: text content changes, font changes, or locale changes. Re-layout on every width change.

## Use Case 2: Get Actual Line Contents

When you need the actual text of each line — for canvas rendering, SVG, WebGL, custom rendering:

```ts
import { prepareWithSegments, layoutWithLines } from '@chenglou/pretext'

const prepared = prepareWithSegments('Hello world, this is a test', '16px Inter')
const { lines, height, lineCount } = layoutWithLines(prepared, 120, 20)

for (let i = 0; i < lines.length; i++) {
  ctx.fillText(lines[i].text, 0, i * 20)
}
```

Use `prepareWithSegments` (not `prepare`) when you need line contents — it retains segment boundaries needed for line extraction.

Each `LayoutLine` has:
- `text` — rendered text of this line
- `width` — measured width
- `start` / `end` — cursor positions into the segment stream

## Use Case 3: Shrinkwrap / Find Tightest Width

Find the narrowest container width that still fits without adding extra lines — useful for chat bubbles, tags, inline labels:

```ts
import { prepareWithSegments, walkLineRanges, layout } from '@chenglou/pretext'

const prepared = prepareWithSegments(text, font)
const baseline = layout(prepared, maxWidth, lineHeight).lineCount

// Binary search for tightest width with same line count
let lo = 1, hi = Math.ceil(maxWidth)
while (lo < hi) {
  const mid = Math.floor((lo + hi) / 2)
  if (layout(prepared, mid, lineHeight).lineCount <= baseline) hi = mid
  else lo = mid + 1
}

// Measure actual max line width at tightest width
let maxLineWidth = 0
walkLineRanges(prepared, lo, line => {
  if (line.width > maxLineWidth) maxLineWidth = line.width
})
// maxLineWidth is the shrinkwrap width
```

Binary search works here because `layout()` is so fast (~0.09ms/500 texts) that calling it dozens of times is negligible.

## Use Case 4: Variable-Width Streaming Layout

When each line can have a different width — text wrapping around images, multi-column flow, obstacle-aware layout:

```ts
import { prepareWithSegments, layoutNextLine, type LayoutCursor } from '@chenglou/pretext'

const prepared = prepareWithSegments(longText, '16px Inter')
let cursor: LayoutCursor = { segmentIndex: 0, graphemeIndex: 0 }
let y = 0

while (true) {
  const width = y < imageBottom ? columnWidth - imageWidth : columnWidth
  const line = layoutNextLine(prepared, cursor, width)
  if (line === null) break

  ctx.fillText(line.text, x, y)
  cursor = line.end
  y += lineHeight
}
```

`layoutNextLine` returns `null` when all text has been consumed. This streaming approach avoids needing to know all widths upfront — compute each width on the fly based on layout state.

## Pre-Wrap Mode

For preserving spaces, tabs, and hard breaks (like `<textarea>`):

```ts
const prepared = prepare(textareaValue, '16px Inter', { whiteSpace: 'pre-wrap' })
const { height } = layout(prepared, textareaWidth, 20)
```

In pre-wrap mode: spaces are preserved, `\t` aligns to tab stops (tab-size: 8), `\n` forces line breaks.

## Locale and Cache Management

```ts
import { setLocale, clearCache } from '@chenglou/pretext'

setLocale('th')
const thaiText = prepare('ภาษาไทย', '16px Noto Sans Thai')

setLocale(undefined)
clearCache()
```

Call `setLocale` before `prepare()` for proper word segmentation in languages like Thai/Lao that don't use spaces between words. `setLocale()` also clears caches internally.

## Common Patterns

### Virtualized List with Accurate Heights

```ts
const items = texts.map(text => ({
  text,
  prepared: prepare(text, '14px Inter'),
}))

function getItemHeights(width: number): number[] {
  return items.map(item => layout(item.prepared, width, 18).height)
}
```

### Prevent Layout Shift on Text Load

```ts
const prepared = prepare(placeholderText, font)
const { height } = layout(prepared, containerWidth, lineHeight)
container.style.minHeight = `${height}px`
```

## Limitations

Pretext targets the common CSS text configuration: `white-space: normal`, `word-break: normal`, `overflow-wrap: break-word`, `line-break: auto`. It does **not** support:
- CSS properties: `break-all`, `keep-all`, `strict`, `loose`, `anywhere`
- Text alignment (left/center/right/justify) — alignment is a separate layer
- `system-ui` font on macOS (measurement drift vs DOM)

## Verification

After integration, verify correctness by comparing Pretext measurements against actual DOM rendering:

```ts
const prepared = prepare(testText, '16px Inter')
const { height } = layout(prepared, 300, 20)

const el = document.createElement('div')
el.style.cssText = 'font: 16px Inter; width: 300px; line-height: 20px; position: absolute; visibility: hidden;'
el.textContent = testText
document.body.appendChild(el)
const domHeight = el.getBoundingClientRect().height
document.body.removeChild(el)

console.assert(Math.abs(height - domHeight) < 2, `Pretext ${height} vs DOM ${domHeight}`)
```

Tolerance of ~1-2px is expected due to subpixel rounding differences.
