# JetBrains Mono Font

## Overview

JetBrains Mono is a free, open-source monospaced font designed for working in terminals. While it was originally made for developers, it's equally well suited for sysadmins spending hours in SSH sessions, reading logs, and monitoring systems. It features increased x-height for better readability at small sizes and distinct character shapes to reduce ambiguity (e.g. `0` vs `O`, `1` vs `l`) — which matters when you're parsing IPs, config values, or hex strings.

- Website: https://www.jetbrains.com/lp/mono/
- License: SIL Open Font License 1.1

## The Line Height Issue

JetBrains Mono ships with a generous default line height. Out of the box, many terminals and editors will render it with noticeable vertical gaps between lines. This makes text look stretched and wastes screen space.

**To make JetBrains Mono look nicer, you almost always need to reduce the line height in your application.**

Without reducing line height, the font feels too "airy" and you lose visible lines of code on screen. Tightening the spacing makes it feel compact and clean — which is how it's meant to look.

## macOS Terminal.app

Terminal.app does **not** support line height adjustment. You can set the font via Settings → Profiles → Text → Font, but there is no vertical spacing control. This is why iTerm2 is recommended instead.

## iTerm2 Configuration

1. Preferences → Profiles → Text
2. Set font to JetBrains Mono
3. Set **vertical spacing** to `90%` (or lower to taste)

Start with font size 13 and 90% vertical spacing, then adjust to preference.

![iterm2]()

## Variants

| Variant | Description |
|---------|-------------|
| **JetBrains Mono** | Standard version with ligatures |
| **JetBrains Mono NL** | No Ligatures — same metrics, no ligature rendering |

## Tips

- The font is optimized for screen readability — it looks great at 12–14px
- If column alignment breaks, you've gone too far with letter-spacing tweaks (don't do that for code)
