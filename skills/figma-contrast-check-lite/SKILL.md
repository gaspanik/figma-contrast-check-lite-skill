---
name: figma-contrast-check-lite
description: Lightweight version of the "figma-contrast-check" skill. Accepts a Figma FRAME, SECTION, COMPONENT, or COMPONENT_SET URL, walks the node tree, resolves each text node's effective foreground and background color (including composited ancestor/sibling fills), and checks WCAG AA color contrast ratios (normal/large text). Reports pass/fail per node with sample text and resolved colors — read-only, no fixes applied. Use whenever the user wants a quick color-accessibility / contrast / WCAG check on a Figma design, mentions "コントラストチェック", "色のアクセシビリティ", "WCAG", or wants to verify color choices before implementation. For AAA conformance, non-text UI element checks, or auto-fixing failing colors, use the full `figma-contrast-check` skill instead. Pairs well as a follow-up to figma-audit (structural AI-readiness) — the two cover different axes of the same "is this frame ready to hand off" question.
argument-hint: <figma-frame-url>
allowed-tools: mcp__plugin_figma_figma__get_metadata, mcp__plugin_figma_figma__use_figma, mcp__plugin_figma_figma__get_screenshot
---

# figma-contrast-check-lite — WCAG AA Contrast Check (Lite)

Analyze the Figma URL provided in `$ARGUMENTS`, resolve the real foreground/background color pairs behind every piece of visible text, and check them against the WCAG AA contrast requirement. This is a read-only check-and-report skill — it never modifies the Figma file.

**Output language:** Respond in the same language the user is using in this conversation.

## Why this exists

Color accessibility gets decided in the design tool, not in code — by the time a page is implemented, the color choices are usually treated as fixed and any contrast fix becomes a design-vs-code back-and-forth. This skill checks the same WCAG math against the Figma source of truth, before implementation. It complements `figma-audit` (structural AI-readiness): running this right after `figma-audit` covers both "can AI implement this accurately" and "will the result be usable" in one pass.

This is the lite edition: WCAG AA only, text contrast only, report only. For AAA conformance, an advisory non-text UI element pass (borders/icons), or confirmation-gated auto-fix of failing colors, use the full `figma-contrast-check` skill.

## Scope

**What this checks:** WCAG 2.x text contrast (SC 1.4.3) for every visible text node in the target tree, against its *effective* background — not just the immediate parent's fill, but the actually-composited result of whatever fills sit behind it (covering sibling shapes, ancestor fills, opacity stacking). See [references/wcag-contrast-algorithm.md](references/wcag-contrast-algorithm.md) for the full resolution algorithm and formulas — read it before scanning, don't improvise a simpler version.

**What this does NOT do:** AAA-level checking, non-text UI element contrast, or any write operation. It never changes the file.

**Target node types:** FRAME, SECTION, COMPONENT, or COMPONENT_SET. This skill does NOT skip `INSTANCE` subtrees during the scan — an instance renders real, visible text that a user will actually read, so it needs checking regardless of where it lives.

Any other node type (e.g. a bare `TEXT` or `RECTANGLE`, or a `PAGE`): ask the user for a valid frame/section/component URL.

---

## Step 1: Parse the URL

Extract from `$ARGUMENTS`:

- `figma.com/design/:fileKey/...?node-id=X-Y` → `fileKey` and `nodeId` (convert `X-Y` to `X:Y`)

If extraction fails, ask the user for a valid Figma URL.

---

## Step 2: Fetch metadata and check scope

Call `get_metadata` for the target node.

- `FRAME`, `SECTION`, `COMPONENT`, or `COMPONENT_SET` → continue
- anything else → ask for a valid URL of one of those types

No further questions are needed — this lite version always checks WCAG AA, text only.

---

## Step 3: Scan text nodes

Walk the node tree depth-first from the root (do not skip `INSTANCE`). Skip nodes with `visible === false` or resolved opacity of 0.

For every `TEXT` node, call `node.getStyledTextSegments(['fontSize', 'fontName', 'fills'])`. Each segment is an independent check target: record its sample text (truncate to ~40 chars for the report), `fontSize`, weight/style (from `fontName.style`), and fill.

For each segment, resolve:
- **Foreground color** — the segment's `SOLID` fill (Section 4 of the reference doc). If non-solid, mark `⚠️ manual check`.
- **Effective background color** — full ancestor/sibling compositing walk (Section 5 of the reference doc). If the walk hits a gradient/image with no opaque layer behind it, mark `⚠️ manual check`.

This is a read-only `use_figma` call.

---

## Step 4: Compute ratios and classify

For each resolved segment, using the formulas in the reference doc:

1. Classify large-vs-normal text from `fontSize` + weight.
2. Compute the contrast ratio between foreground and effective background.
3. Look up the required minimum for AA + size class (4.5:1 normal, 3:1 large).
4. Verdict: ✅ Pass (ratio ≥ required), ❌ Fail (ratio < required), or ⚠️ carried over from Step 3 for anything marked manual-check.

---

## Step 5: Report

If zero text nodes were found, say so and stop.

Otherwise output:

---

### Contrast check report (WCAG AA)

| # | Layer | Sample text | Size / Weight | FG | BG (resolved) | Ratio | Required | Verdict |
|---|-------|-------------|----------------|----|-----------------|-------|----------|---------|
| 1 | Hero > Heading | "Build faster with..." | 32px / Bold | #1A1A1A | #FFFFFF | 16.1:1 | 3:1 (large) | ✅ Pass |
| 2 | Card > Body | "Lorem ipsum dolor..." | 14px / Regular | #9CA3AF | #FFFFFF | 2.8:1 | 4.5:1 | ❌ Fail |
| 3 | Footer > Legal | "© 2026 Example Inc." | 12px / Regular | (gradient text) | — | — | — | ⚠️ Manual check — non-solid text fill |

**Summary: N passed / N failed / N need manual check** (out of N total)

If any ❌ Fail rows exist, mention that the full `figma-contrast-check` skill can auto-fix them (with confirmation) once installed.

---

## Error handling

- If fileKey or nodeId cannot be parsed: ask for a valid URL
- If the node is not FRAME/SECTION/COMPONENT/COMPONENT_SET: ask for a valid URL of one of those types
- If `get_metadata` returns an error: report it and ask the user to verify Figma file access
- If `use_figma` fails during the scan: report the failure inline, don't retry blindly, and report results for whatever was successfully scanned before the failure
- If the background resolution walk reaches the tree root with no opaque fill found: use the page background and label that row `assumed page background` in the report, since the true backdrop may live outside the URL's target node
