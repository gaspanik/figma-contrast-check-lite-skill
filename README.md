# figma-contrast-check-lite — WCAG AA Contrast Check for Figma Frames (Lite)

A Claude Code skill that walks a Figma FRAME, SECTION, COMPONENT, or COMPONENT_SET, resolves each text node's *effective* foreground and background color (compositing through covering siblings, ancestor fills, and opacity), and checks the result against the WCAG AA contrast requirement — before the design ever reaches code.

This is the lite edition: WCAG AA only, text contrast only, report only. It never modifies the Figma file.

---

## What this is

Color accessibility decisions are made in the design tool, not in code. By the time a page is implemented, a contrast fix usually turns into a design-vs-code back-and-forth. This skill runs the WCAG AA math directly against the Figma source of truth and reports a per-text-node pass/fail with the actual resolved colors and ratio.

```
Figma FRAME/SECTION/COMPONENT URL
        │
        ▼
Walk tree, resolve effective FG/BG per text segment
  (covering siblings → ancestor fills → alpha compositing → page bg)
        │
        ▼
Compute WCAG AA ratio, classify large/normal text, compare to threshold
        │
        ▼
Report table (Pass / Fail / Manual check)
```

This skill does **not** skip `INSTANCE` subtrees when scanning — an instance renders real visible text that needs checking wherever it lives.

---

## Scope

- **In scope:** WCAG 1.4.3 text contrast at the AA level, computed against the actually-composited background (not just the immediate parent's fill).
- **Out of scope:** AAA conformance, non-text UI element contrast (borders/icons), and any write/fix operation. For those, see the full `figma-contrast-check` skill.
- Pairs well with `figma-audit` (structural AI-readiness) as a second, color-accessibility pass over the same frame.

---

## Repo structure

```
skills/
  figma-contrast-check-lite/
    SKILL.md
    LICENSE
    references/
      wcag-contrast-algorithm.md   — luminance/contrast formulas, background-resolution algorithm
```

---

## Getting started

**1. Clone this repo**

```bash
git clone https://github.com/gaspanik/figma-contrast-check-lite-skill
```

**2. Install the skill into Claude Code**

```bash
cp -r skills/figma-contrast-check-lite ~/.claude/skills/
```

**3. Verify the Figma MCP is connected**

This skill calls `use_figma` in read-only mode and requires the official Figma MCP server.

**4. Run the skill**

```
/figma-contrast-check-lite https://www.figma.com/design/<fileKey>/...
```

```
このFigmaフレームの色のコントラストをWCAG基準でチェックして: https://www.figma.com/design/...
```

```
Check this frame's color contrast against WCAG AA: https://www.figma.com/design/...
```

---

## When the skill is triggered automatically

The skill description instructs Claude to use it whenever the user wants a quick color-accessibility/contrast/WCAG check on a Figma design. It pairs naturally as a follow-up to `figma-audit`.

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Claude Code](https://claude.ai/code) + [Figma MCP](https://github.com/figma/mcp-server-guide)
