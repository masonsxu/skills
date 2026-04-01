---
name: pretext-integration
description: |
  How to integrate and use @chenglou/pretext — a pure JS/TS library for fast, DOM-free multiline text measurement and layout. Use this skill whenever the user mentions text measurement, text height calculation, text layout, line breaking, word wrapping, canvas text rendering, virtualized text, shrinkwrap text, custom text layout, or anything related to measuring or laying out text in the browser without DOM reflow. Also trigger when the user mentions pretext, @chenglou/pretext, or wants to avoid getBoundingClientRect / offsetHeight for text sizing. This covers both the simple height-measurement use case and the advanced manual-layout use case (canvas rendering, SVG text, text-around-obstacles, shrinkwrap, streaming line-by-line layout).
---

# Pretext Integration Guide

Pretext is a pure JavaScript/TypeScript library that measures and lays out multiline text without touching the DOM. It uses canvas `measureText` under the hood and caches segment widths so that repeated layout calculations (e.g., on resize) are pure arithmetic — no layout reflow, no DOM reads.

## Core Concept: Two-Phase Design

Everything in Pretext revolves around the split between **expensive preparation** and **cheap layout**:

1. **`prepare()` / `prepareWithSegments()`** — Runs once when text first appears. Segments the text, measures each segment via canvas, caches widths. This is the heavy phase (~19ms for 500 texts on the current benchmark).
2. **`layout()` / `layoutWithLines()`** / etc. — Runs on every resize or width change. Pure arithmetic over cached widths. This is the fast path (~0.09ms for 500 texts).

The key insight: **do not re-run prepare when only the width changes**. Prepare once, layout many times.

## Installation

```sh
npm install @chenglou/pretext
```

## Use Case 1: Measure Paragraph Height (No DOM)

This is the most common use case. You want to know how tall a block of text will be at a given width — for virtualization, layout shift prevention, overflow detection, etc.

```ts
import { prepare, layout } from '@chenglou/pretext'

const prepared = prepare('Hello world', '16px Inter')
const { height, lineCount } = layout(prepared, 200, 20) // 200px width, 20px line height
```

### The `font` parameter

The `font` string must match what you'd set on a canvas context (`ctx.font`). It uses the same CSS font shorthand format:

```ts
'16px Inter'
'bold 14px "Helvetica Neue", Helvetica, sans-serif'
'italic 20px Georgia, serif'
```

**Important**: This font string must be synchronized with your CSS font declaration for the text you're measuring. If your CSS says `font: 16px Inter` then pass `'16px Inter'` to prepare. Mismatched fonts mean mismatched measurements.

**Caveat**: Avoid `system-ui` on macOS — canvas and DOM can resolve different font variants, causing measurement drift. Use named fonts like `Inter`, `Helvetica`, etc.

### The `lineHeight` parameter

Must match your CSS `line-height`. If your CSS has `line-height: 24px`, pass `24` to layout.

### When to re-prepare vs re-layout

```ts
const prepared = prepare(text, font) // once, when text or font changes

function onResize(newWidth: number) {
  const { height, lineCount } = layout(prepared, newWidth, lineHeight) // fast, no DOM
}
```

Re-prepare only when: text content changes, font changes, or locale changes. Re-layout on every width change.

## Use Case 2: Get Actual Line Contents

When you need the actual text of each line (for canvas rendering, SVG, WebGL, custom rendering):

```ts
import { prepareWithSegments, layoutWithLines } from '@chenglou/pretext'

const prepared = prepareWithSegments('Hello world, this is a test', '16px Inter')
const { lines, height, lineCount } = layoutWithLines(prepared, 120, 20)

for (let i = 0; i < lines.length; i++) {
  ctx.fillText(lines[i].text, 0, i * 20)
}
```

Each `LayoutLine` has:
- `text` — the rendered text content of this line (e.g., `'Hello world,'`)
- `width` — the measured width of this line
- `start` / `end` — cursor positions into the segment stream

## Use Case 3: Shrinkwrap / Find Tightest Width

Find the narrowest container width that still fits the text without adding extra lines:

```ts
import { prepareWithSegments, walkLineRanges, layout } from '@chenglou/pretext'

const prepared = prepareWithSegments(text, font)

// Get the baseline line count at your max width
const baseline = layout(prepared, maxWidth, lineHeight).lineCount

// Binary search for the tightest width with the same line count
let lo = 1
let hi = Math.ceil(maxWidth)
while (lo < hi) {
  const mid = Math.floor((lo + hi) / 2)
  if (layout(prepared, mid, lineHeight).lineCount <= baseline) {
    hi = mid
  } else {
    lo = mid + 1
  }
}

// Now lo is the tightest width — use walkLineRanges to get the actual max line width
let maxLineWidth = 0
walkLineRanges(prepared, lo, line => {
  if (line.width > maxLineWidth) maxLineWidth = line.width
})
// maxLineWidth is the shrinkwrap width
```

## Use Case 4: Variable-Width Streaming Layout

When each line can have a different width (text wrapping around images, multi-column flow, obstacle-aware layout):

```ts
import { prepareWithSegments, layoutNextLine, type LayoutCursor } from '@chenglou/pretext'

const prepared = prepareWithSegments(longText, '16px Inter')
let cursor: LayoutCursor = { segmentIndex: 0, graphemeIndex: 0 }
let y = 0

while (true) {
  // Width changes per line — e.g., narrower beside a floated image
  const width = y < imageBottom ? columnWidth - imageWidth : columnWidth
  const line = layoutNextLine(prepared, cursor, width)
  if (line === null) break

  ctx.fillText(line.text, x, y)
  cursor = line.end
  y += lineHeight
}
```

## Pre-Wrap Mode (Textarea-Like Text)

For preserving spaces, tabs, and hard breaks (like a `<textarea>`):

```ts
const prepared = prepare(textareaValue, '16px Inter', { whiteSpace: 'pre-wrap' })
const { height } = layout(prepared, textareaWidth, 20)
```

In pre-wrap mode:
- Ordinary spaces are preserved (not collapsed)
- `\t` tabs align to browser-style tab stops (tab-size: 8)
- `\n` creates forced line breaks
- Other wrapping rules stay the same (`word-break: normal`, `overflow-wrap: break-word`, `line-break: auto`)

## Locale and Cache Management

```ts
import { setLocale, clearCache } from '@chenglou/pretext'

// Set locale before prepare() for proper word segmentation (Thai, Lao, etc.)
setLocale('th')
const thaiText = prepare('ภาษาไทย', '16px Noto Sans Thai')

// Reset to default locale
setLocale(undefined) // or setLocale() — same effect

// Clear all internal caches if cycling through many fonts/texts
clearCache()
```

`setLocale()` also clears caches internally. Setting locale does not affect already-prepared text.

## Common Patterns

### Virtualized List with Accurate Heights

```ts
// Prepare all visible items once
const items = texts.map(text => ({
  text,
  prepared: prepare(text, '14px Inter'),
}))

// On resize, recalculate all heights cheaply
function getItemHeights(width: number): number[] {
  return items.map(item => layout(item.prepared, width, 18).height)
}
```

### Chat Bubble Shrinkwrap

```ts
// Prepare once per message
const prepared = prepareWithSegments(message, '15px "Helvetica Neue"')

// Find tightest width that keeps the same line count as the max bubble width
const maxWidth = chatWidth * 0.8 - padding * 2
const targetLines = layout(prepared, maxWidth, 20).lineCount

let lo = 1, hi = Math.ceil(maxWidth)
while (lo < hi) {
  const mid = Math.floor((lo + hi) / 2)
  if (layout(prepared, mid, 20).lineCount <= targetLines) hi = mid
  else lo = mid + 1
}

// Measure actual max line width at tightest width
let tightWidth = 0
walkLineRanges(prepared, lo, line => {
  if (line.width > tightWidth) tightWidth = line.width
})

// Final bubble width = content + padding
const bubbleWidth = Math.ceil(tightWidth) + padding * 2
```

### Prevent Layout Shift on Text Load

```ts
// Before text loads, reserve space using predicted height
const prepared = prepare(placeholderText, font)
const { height } = layout(prepared, containerWidth, lineHeight)
container.style.minHeight = `${height}px`

// After text loads, the actual content fills exactly the reserved space
```

## Performance Notes

- `prepare()` cost scales with text length and complexity. For a batch of 500 varied texts, expect ~19ms total.
- `layout()` is the hot path at ~0.09ms per 500-text batch. It's safe to call on every frame during resize animations.
- Segment metrics are cached globally (`Map<font, Map<segment, metrics>>`). Multiple `prepare()` calls with the same font share cached measurements.
- For very large numbers of texts with different fonts, call `clearCache()` periodically to release memory.

## What Pretext Does NOT Do

- It is not a full font rendering engine. It targets the common CSS text configuration: `white-space: normal`, `word-break: normal`, `overflow-wrap: break-word`, `line-break: auto`.
- CSS properties like `break-all`, `keep-all`, `strict`, `loose`, `anywhere` are not yet supported.
- `system-ui` font gives inaccurate measurements on macOS. Always use named fonts.
- Text alignment (left/center/right/justify) is not handled — Pretext measures and breaks lines, but alignment is a separate concern you layer on top.
