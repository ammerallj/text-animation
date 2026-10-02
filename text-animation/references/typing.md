# Typing (typewriter) mode

Characters appear instantly one at a time with a cursor, at a human, slightly uneven pace — not the slide-up reveal. Built for screen recordings, case studies, and motion pieces.

How it works (both platforms):
- The whole string is always rendered: typed part visible, untyped part `color: transparent`. Layout, wrapping, and kerning are final from the first frame, so nothing reflows while typing.
- The cursor is a zero-width inline element between the two parts, so it never pushes text.
- Cursor is solid while typing and blinks (1s, hard on/off) while idle — before the start and after the end, like a real text field.
- Pacing is "humanized": each keystroke is `msPerChar` × 0.6–1.4, spaces are a bit faster, and there are pauses after `, ; :` (+180ms), `. ! ?` (+320ms), and line breaks (+400ms).
- The jitter comes from a seeded random generator, so **every take is identical** — important when you're re-recording or syncing to other motion. Change `seed` for a different rhythm.

Shared props:

```
text: string               // "\n" = line break
msPerChar: number   = 55   // 40 feels fast and confident, 80 feels deliberate
startDelayMs: number = 400 // cursor blinks this long before typing starts
humanize: boolean   = true // false = perfectly even timing
seed: number        = 1
play: boolean       = true // false → reset to empty with blinking cursor
cursor: boolean     = true
cursorColor?: string       // defaults to the text color
onComplete?: () => void
```

Replay = change the component's `key`.

## Shared timing helper (include in either component file)

```ts
function splitGraphemes(s: string): string[] {
  const Segmenter = (Intl as any)?.Segmenter
  if (Segmenter) return Array.from(new Segmenter(undefined, { granularity: "grapheme" }).segment(s), (x: any) => x.segment)
  return Array.from(s)
}

// Seeded PRNG so every take has the same rhythm.
function mulberry32(seed: number) {
  return () => {
    seed |= 0
    seed = (seed + 0x6d2b79f5) | 0
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}

// gaps[i] = wait after typing chars[i]
function typingGaps(chars: string[], msPerChar: number, humanize: boolean, seed: number): number[] {
  const rand = mulberry32(seed)
  return chars.map((c) => {
    let ms = humanize ? msPerChar * (0.6 + rand() * 0.8) : msPerChar
    if (c === " ") ms *= 0.8
    if (humanize) {
      if (",;:".includes(c)) ms += 180
      else if (".!?".includes(c)) ms += 320
      else if (c === "\n") ms += 400
    }
    return Math.round(ms)
  })
}
```

## React / Next.js web

```tsx
"use client"

import { useEffect, useMemo, useState, type CSSProperties, type ElementType } from "react"

// ...splitGraphemes, mulberry32, typingGaps from above...

const CSS = `
.tw-caret{position:relative;display:inline-block;width:0;height:1em;vertical-align:-0.12em}
.tw-caret::after{content:"";position:absolute;left:1px;top:0;bottom:0;width:max(2px,.06em);
  background:var(--tw-caret,currentColor);animation:tw-blink 1s steps(1) infinite}
.tw-caret.tw-solid::after{animation:none}
@keyframes tw-blink{50%{opacity:0}}
`

type Props = {
  text: string
  as?: ElementType
  msPerChar?: number
  startDelayMs?: number
  humanize?: boolean
  seed?: number
  play?: boolean
  cursor?: boolean
  cursorColor?: string
  onComplete?: () => void
  className?: string
  style?: CSSProperties
}

export function TypewriterText({
  text,
  as: Tag = "p",
  msPerChar = 55,
  startDelayMs = 400,
  humanize = true,
  seed = 1,
  play = true,
  cursor = true,
  cursorColor,
  onComplete,
  className,
  style,
}: Props) {
  const chars = useMemo(() => splitGraphemes(text), [text])
  const gaps = useMemo(() => typingGaps(chars, msPerChar, humanize, seed), [chars, msPerChar, humanize, seed])
  const [count, setCount] = useState(0)
  const [typing, setTyping] = useState(false)

  useEffect(() => {
    setCount(0)
    setTyping(false)
    if (!play) return
    let i = 0
    let t: number
    const step = () => {
      i++
      setCount(i)
      if (i >= chars.length) {
        setTyping(false)
        onComplete?.()
        return
      }
      t = window.setTimeout(step, gaps[i - 1])
    }
    t = window.setTimeout(() => {
      setTyping(true)
      step()
    }, startDelayMs)
    return () => window.clearTimeout(t)
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [play, chars, gaps, startDelayMs])

  return (
    <Tag className={className} style={{ whiteSpace: "pre-wrap", ...style }}>
      <style>{CSS}</style>
      {chars.slice(0, count).join("")}
      {cursor && (
        <span
          className={`tw-caret${typing ? " tw-solid" : ""}`}
          style={cursorColor ? ({ "--tw-caret": cursorColor } as CSSProperties) : undefined}
        />
      )}
      <span style={{ color: "transparent" }}>{chars.slice(count).join("")}</span>
    </Tag>
  )
}
```

## React Native

```tsx
import React, { useEffect, useMemo, useRef, useState } from 'react';
import { Animated, StyleProp, StyleSheet, Text, TextStyle } from 'react-native';

// ...splitGraphemes, mulberry32, typingGaps from above...

type Props = {
  text: string;
  msPerChar?: number;
  startDelayMs?: number;
  humanize?: boolean;
  seed?: number;
  play?: boolean;
  cursor?: boolean;
  cursorColor?: string;
  onComplete?: () => void;
  textStyle?: StyleProp<TextStyle>;
};

export default function TypewriterText({
  text,
  msPerChar = 55,
  startDelayMs = 400,
  humanize = true,
  seed = 1,
  play = true,
  cursor = true,
  cursorColor,
  onComplete,
  textStyle,
}: Props) {
  const chars = useMemo(() => splitGraphemes(text), [text]);
  const gaps = useMemo(() => typingGaps(chars, msPerChar, humanize, seed), [chars, msPerChar, humanize, seed]);
  const [count, setCount] = useState(0);
  const [typing, setTyping] = useState(false);
  const blink = useRef(new Animated.Value(1)).current;

  useEffect(() => {
    setCount(0);
    setTyping(false);
    if (!play) return;
    let i = 0;
    let t: ReturnType<typeof setTimeout>;
    const step = () => {
      i++;
      setCount(i);
      if (i >= chars.length) {
        setTyping(false);
        onComplete?.();
        return;
      }
      t = setTimeout(step, gaps[i - 1]);
    };
    t = setTimeout(() => {
      setTyping(true);
      step();
    }, startDelayMs);
    return () => clearTimeout(t);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [play, chars, gaps, startDelayMs]);

  // Solid while typing, hard on/off blink while idle.
  useEffect(() => {
    if (typing) {
      blink.setValue(1);
      return;
    }
    const loop = Animated.loop(
      Animated.sequence([
        Animated.timing(blink, { toValue: 0, duration: 0, delay: 500, useNativeDriver: true }),
        Animated.timing(blink, { toValue: 1, duration: 0, delay: 500, useNativeDriver: true }),
      ]),
    );
    loop.start();
    return () => loop.stop();
  }, [typing, blink]);

  const flat = StyleSheet.flatten(textStyle) ?? {};
  const fontSize = flat.fontSize ?? 14;

  return (
    <Text style={textStyle}>
      {chars.slice(0, count).join('')}
      {cursor && (
        <Animated.View
          style={{
            width: 2,
            marginRight: -2, // zero net width so text never shifts
            height: fontSize,
            backgroundColor: cursorColor ?? (flat.color as string) ?? '#000',
            opacity: blink,
            transform: [{ translateY: fontSize * 0.15 }],
          }}
        />
      )}
      <Text style={{ color: 'transparent' }}>{chars.slice(count).join('')}</Text>
    </Text>
  );
}
```

Inline `View`s inside `Text` sit on the baseline; if the cursor looks too high or low for a particular font, adjust the `translateY` factor (0.1–0.25).

## Tips for recordings
- Use `startDelayMs` of ~600–1000 so the viewer sees the blinking cursor before typing begins.
- Keep the same `seed` across takes; try a few seeds and keep the rhythm you like best.
- For a "typed then revealed" sequence, chain with `onComplete` (e.g., type a prompt, then reveal the response with `SplitTextReveal`).

---

# CLI mode (terminal typing)

Imitates a command line: a prompt, a command typed at keyboard pace, Enter, then output that prints line by line. Use it when the user says "CLI", "terminal", "command line", "like Claude Code", or wants a developer-tool feel.

What makes it read as a real terminal rather than a typewriter:
- **Monospace font** (`ui-monospace, SF Mono, Menlo, monospace`), left-aligned, no wrapping surprises.
- **Prompt prefix** (`>`, `$`, or `❯`) in a dimmed or accent color; it appears instantly, only the command is typed.
- **Block cursor** (a filled rectangle one character wide), not a thin bar. Solid while typing, blinks when idle.
- **Faster, steadier typing** than the prose mode: `msPerChar` 30–45, `humanize` on but with smaller jitter, no long punctuation pauses (commands aren't sentences).
- **A beat before Enter** (~350ms), then the cursor leaves the command line and output starts.
- **Output prints in lines, not letters**: each line appears whole after its own delay (80–400ms). Slow steps get a spinner (`⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏`, 80ms per frame) that resolves to `✓` or `✗`.
- **Colors carry meaning** (dim for metadata, green for success, red for errors, accent for paths or file names). Keep to 3–4 colors.

## Script format

Drive the whole sequence from one array so it's easy to edit and replay:

```ts
type Step =
  | { type: "type"; prompt?: string; text: string; msPerChar?: number }  // types a command, then "Enter"
  | { type: "print"; text: string; tone?: "dim" | "ok" | "err" | "accent"; delayMs?: number }
  | { type: "spinner"; text: string; durationMs: number; result?: "ok" | "err" }
  | { type: "pause"; ms: number }
  | { type: "clear" }
```

Render the finished lines from state, plus one "live" line (the line currently being typed, with the block cursor, or the spinner). Run the script with a single async loop (`for (const step of script) await run(step)`) and a cancel token so replay (changing `key`) stops the old run cleanly. Reuse `typingGaps` from above for the `type` step, passing `humanize` jitter but zeroing the punctuation pauses.

## Web demo structure

```html
<div class="term" role="img" aria-label="Terminal demo">
  <div id="lines"></div>
  <div class="live"><span class="prompt">❯</span> <span id="typed"></span><span class="block"></span></div>
</div>
```

```css
.term{background:#0f1115;color:#e6e6e6;font:14px/1.6 ui-monospace,SFMono-Regular,Menlo,monospace;padding:16px 18px;border-radius:12px}
.block{display:inline-block;width:.6em;height:1.15em;vertical-align:text-bottom;background:#e6e6e6}
.block.idle{animation:blink 1s steps(1) infinite}
@keyframes blink{50%{opacity:0}}
```

## React Native notes
Use a `ScrollView` with `scrollToEnd` when new lines print, a `Text` with `fontFamily: Platform.select({ ios: "Menlo", android: "monospace" })`, and render the block cursor as an inline-sized `View` (width = character width ≈ `fontSize * 0.6`).

## For recordings
Pin the pacing: fixed `seed`, explicit `delayMs` on every print, and a final `pause` so the last frame holds for a moment before the take ends.

## Themes (dark and light)

CLI mode ships with two themes; default to dark and switch to light when the user asks for "white", "light", or the surrounding page is light. Drive the whole terminal from CSS variables / a theme object so a single `theme` prop (`"dark" | "light"`) swaps everything, including the cursor.

| Token | Dark | Light |
|---|---|---|
| background | `#0f1115` | `#ffffff` |
| border | none | `0.5px solid #d0d7de` (a white terminal needs an edge or it vanishes on a white page) |
| text | `#e6e6e6` | `#1f2328` |
| dim | `#7d8590` | `#6e7781` |
| prompt | `#d2a8ff` | `#8250df` |
| accent (paths, spinner) | `#79c0ff` | `#0969da` |
| ok | `#56d364` | `#1a7f37` |
| err | `#ff7b72` | `#cf222e` |
| cursor | `#e6e6e6` | `#1f2328` |
| selection / highlight | `#264f78` | `#b6e3ff` |

Light-theme rules, which are easy to get wrong:
- Don't just invert colors. Dark-theme greens and blues are too pale on white; use the darker light-theme values above (they pass 4.5:1 against white).
- Keep the block cursor dark and fully opaque; a pale cursor disappears.
- Add the hairline border and, if it sits on white, a very soft shadow (`0 1px 2px rgba(0,0,0,.06)`), nothing heavier.
- Same fonts, spacing, timing, and radius in both themes, so a recording can cut between them.

Web: define the tokens as `--t-bg`, `--t-fg`, `--t-dim`, `--t-prompt`, `--t-acc`, `--t-ok`, `--t-err`, `--t-cursor`, set them on `.term[data-theme="dark"]` / `.term[data-theme="light"]`, and color everything with `var(--t-*)`.

React Native: keep a `THEMES = { dark: {...}, light: {...} }` object with the same keys and pass the active one into styles; offer a `theme` prop that defaults to `"dark"`.

## Window chrome (wrapper UI)

An optional frame around the CLI so it reads as a real app window. Prop: `chrome: "macos" | "tabs" | "minimal" | "none"` (default `"macos"`; `"none"` when the user wants the bare terminal).

| Style | What it looks like |
|---|---|
| `macos` | Title bar with red / yellow / green dots at left, centered title (e.g. `~/project — zsh`), dim text |
| `tabs` | Dots, then a tab strip (active tab highlighted, optional second inactive tab), like iTerm / VS Code terminal |
| `minimal` | Slim bar, no dots, left-aligned dim title, hairline divider |
| `none` | No wrapper |

Specs (keep them consistent so recordings look polished):
- **Bar height** 38px (`macos`/`tabs`), 30px (`minimal`); bottom hairline in the theme's border color.
- **Dots** 12px circles, 8px gap, 14px from the left: `#ff5f57`, `#febc2e`, `#28c840` (same in both themes; they're brand-recognizable). Add `1px` inner darker ring in light theme: `inset 0 0 0 0.5px rgba(0,0,0,.15)`.
- **Bar background** one step off the body so the bar reads as a separate layer: dark `#161a21` over body `#0f1115`; light `#f6f8fa` over body `#ffffff`.
- **Title** 13px, system sans (not monospace), dim color, centered, truncating with an ellipsis.
- **Window** radius 12px, clipped (`overflow: hidden`) so the bar and body share corners; one soft outer shadow only on light theme or when floating on a colored backdrop (`0 10px 30px rgba(0,0,0,.18)`).
- **Body** keeps the CLI padding (18px 20px); the chrome adds no extra spacing.
- The dots are decorative: not interactive, `aria-hidden`.
- Optional **backdrop**: a flat solid or subtle two-tone fill behind the window with 32px padding, for case-study hero shots. No gradients unless the user asks.

Structure:

```html
<div class="win" data-theme="dark">
  <div class="bar">
    <span class="dots"><i></i><i></i><i></i></span>
    <span class="title">~/accessibility-kb — zsh</span>
  </div>
  <div class="term">…CLI body…</div>
</div>
```

React Native: render the bar as a `View` (`flexDirection: 'row'`, `height: 38`) with three 12px circular `View`s and a centered `Text`, inside a rounded `overflow: 'hidden'` container; the title `Text` uses the system font, not the monospace one.
