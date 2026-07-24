# WCAG contrast algorithm (lite)

Goal: for every visible text node in the target tree, resolve its effective foreground color and effective background color, compute the WCAG contrast ratio, and classify it against WCAG AA.

## 1. Relative luminance and contrast ratio (WCAG 2.x formula)

```js
function srgbToLinear(c) {
  // c is 0-1
  return c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4);
}

function relativeLuminance({ r, g, b }) {
  // r, g, b are 0-1
  const R = srgbToLinear(r), G = srgbToLinear(g), B = srgbToLinear(b);
  return 0.2126 * R + 0.7152 * G + 0.0722 * B;
}

function contrastRatio(fg, bg) {
  const L1 = relativeLuminance(fg);
  const L2 = relativeLuminance(bg);
  const lighter = Math.max(L1, L2);
  const darker = Math.min(L1, L2);
  return (lighter + 0.05) / (darker + 0.05);
}
```

Round the result to 2 decimal places for display (e.g. `4.52:1`), but compare against thresholds using the unrounded value — a ratio of 4.495 must not display-round to 4.50 and then read as a pass.

## 2. Large-text classification

WCAG defines "large text" in points, not pixels. Figma's `fontSize` is in px. Convert using 1pt = 4/3 px (96dpi):

```js
function isLargeText(fontSizePx, fontWeight) {
  const isBold = fontWeight >= 700; // Figma fontName.style often contains "Bold"/"Semi Bold" — treat weight>=600 as bold-ish if only style string is available, but prefer numeric weight when the font has one
  const boldThresholdPx = 14 * (4 / 3); // 18.67px
  const normalThresholdPx = 18 * (4 / 3); // 24px
  return isBold ? fontSizePx >= boldThresholdPx : fontSizePx >= normalThresholdPx;
}
```

If only a style string is available (`"Bold"`, `"Semi Bold"`, `"Regular"`), treat `Bold`/`Black`/`Heavy`/`ExtraBold` as bold and everything else as not-bold.

## 3. Threshold table (AA only)

| Text type | Minimum ratio |
|-----------|---------------|
| Normal | 4.5:1 |
| Large | 3:1 |

This lite version always checks against WCAG AA. For AAA thresholds, non-text UI element checks (SC 1.4.11), or auto-fix, see the full `figma-contrast-check` skill.

## 4. Resolving effective foreground color

A text node can mix multiple fills across character ranges. Don't read `node.fills` directly — call `node.getStyledTextSegments(['fontSize', 'fontName', 'fills'])` and treat each returned segment as its own check target with its own sample text, size, weight, and fill.

Only `SOLID` fills have a well-defined color. If a segment's fill is `GRADIENT_*` or `IMAGE`, mark that segment `⚠️ manual check — non-solid text fill` rather than guessing a single representative color.

## 5. Resolving effective background color

For a given node, walk outward in this order and stop at the first fully-opaque result:

1. **Preceding siblings within the same parent** that geometrically cover the node's bounding box and sit behind it in z-order (i.e., appear earlier in `parent.children`, since Figma paints children back-to-front). A sibling counts as covering if its absolute bounding box fully contains the text node's bounding box.
2. **The parent node's own `fills`**, if not itself covered by one of its own preceding siblings relative to the grandparent — recurse the same covering-sibling check one level up.
3. **Continue up the ancestor chain**, applying the same two checks, until a fully-opaque SOLID fill is found or the root of the walked tree is reached.
4. If nothing opaque is found by the root, fall back to the page background color (`figma.currentPage.backgrounds`, typically white) — flag this row `assumed page background` in the report, since content is often composed inside a larger design where the true backdrop lives outside the URL's target node.

At each layer, only `visible !== false` nodes with `opacity > 0` count. If a candidate background fill is `SOLID` with `opacity < 1` (or the layer's own `node.opacity < 1`), composite it over whatever is resolved from the next layer down using standard alpha-over-alpha blending, then keep walking outward:

```js
function compositeOver(top, bottom) {
  // top, bottom: { r, g, b, a } all 0-1
  const a = top.a + bottom.a * (1 - top.a);
  if (a === 0) return { r: 0, g: 0, b: 0, a: 0 };
  const blend = (cTop, cBottom) => (cTop * top.a + cBottom * bottom.a * (1 - top.a)) / a;
  return {
    r: blend(top.r, bottom.r),
    g: blend(top.g, bottom.g),
    b: blend(top.b, bottom.b),
    a,
  };
}
```

If any layer in the resolution chain is a `GRADIENT_*` or `IMAGE` fill, stop and mark the node `⚠️ manual check — background is a gradient/image, ratio not computable`. Don't approximate a gradient by sampling one stop.
