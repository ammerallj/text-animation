---
name: text-animation
description: Add text animations to React Native or React / Next.js — (1) split-text reveal, where characters or words slide up one after another from behind an invisible mask like a premium headline lockup, and (2) typing / typewriter, where characters appear one at a time with a blinking cursor and human-feeling pacing. Use this skill whenever the user wants a text reveal, letter-by-letter or word-by-word animation, staggered text entrance, "split text", animated headline, kinetic type, text that slides or rises into view, a typing effect, typewriter text, a blinking cursor, CLI or terminal typing (dark or light theme, with optional macOS-style window chrome), text that types itself out, or an animated text for a prototype, screen recording, case study, or motion piece — even if they don't say "split". Screen reader and reduced-motion handling are built in and can be switched off for recordings.
---

# Text Animation (reveal, typing, CLI)

Two modes:
- **Reveal** — each character slides up out of a clipping slot, staggered. The default. Spec below.
- **Typing** — characters appear instantly with a cursor and human pacing. Read `references/typing.md` for the spec and both components.

If the user says "typing", "typewriter", or "cursor", use typing; if they say "CLI", "terminal", or "command line", use the CLI mode at the end of `references/typing.md` (it has dark and light themes and optional window chrome: `macos`, `tabs`, `minimal`, `none`; default to dark, use light when they say "white" or "light"). If they say "reveal", "slide", "rise", or "split", use reveal. If it's unclear, ask — or offer to chain them (type a prompt, then reveal the response).

A headline-style entrance: each character sits in its own clipping box and slides up into place, staggered from first to last. It reads as text "rising out of a slot."

## The motion spec (keep these unless the user asks otherwise)

| Property | Value | Why |
|---|---|---|
| Unit | characters (default) or words | Characters for short headlines; words for anything longer than ~60 characters, where per-letter staggering gets slow and noisy |
| Start position | translateY 120% of the character's height, clipped | 120% (not 100%) guarantees descenders are fully hidden at the start |
| Duration per unit | 600ms | |
| Easing | cubic-bezier(0.16, 1, 0.3, 1) (ease-out-expo-ish) | Fast launch, long gentle settle — this is most of the "feel" |
| Stagger | 25ms between units | |
| Stagger order | `first` (default), `last`, `center`, `random` | |
| Direction | up (default), or `reverse` to drop in from above (-120%) | |
| Clip padding | 0.15em top, 0.25em bottom, cancelled with equal negative margin | Stops ascenders/descenders (g, y, comma) being cut off once revealed, without changing layout |

Total run time ≈ 600ms + 25ms × (units − 1). Keep it under ~3s; if the text is long enough to blow past that, switch to words or reduce the stagger, and tell the user why.

## Recordings and demos

If the user says the animation is for a screen recording, case study, prototype video, or motion piece, set `respectReducedMotion={false}` (reveal) so the device's Reduce Motion setting can't flatten the animation mid-recording, and skip the accessibility extras that only matter for shipped product (they've said so explicitly). Don't lecture about it. Typing mode has no reduced-motion handling by default, so it's already recording-ready. Mention once, in a line, that the setting exists if they later ship the same screen to real users.

For recordings, determinism matters more than anything: use the same `seed` in typing mode, avoid `random` for `staggerFrom`, and tell the user how to replay a take (change `key`).

## Accessibility requirements for shipped product

Apply these by default; they're the right behavior for anything real users will see. They're optional only when the user has said the work is a recording or demo (see above).

Splitting text into per-letter elements breaks it for assistive tech unless handled deliberately. Every implementation must:

1. **Expose the full string once.** Screen readers should announce "The needs of the many", not "T, h, e…". Put the full text on the wrapper (web: visually hidden text or `aria-label` on a role-appropriate element; RN: `accessible` + `accessibilityLabel` on the wrapper) and hide the split pieces from assistive tech (web: `aria-hidden="true"`; RN: `importantForAccessibility="no-hide-descendants"` and children non-accessible).
2. **Keep semantics.** If the text is a heading, the heading role stays on the wrapper (web: render as `h1`/`h2`; RN: `accessibilityRole="header"`).
3. **Respect reduced motion.** When the user prefers reduced motion (web: `prefers-reduced-motion: reduce`; RN: `AccessibilityInfo.isReduceMotionEnabled()` + change listener), show the text immediately in its final state — no slide. A short opacity fade (≤200ms) is acceptable; movement is not. (WCAG 2.3.3 Animation from Interactions.)
4. **Never leave text hidden.** If the animation can't run (no JS yet, effect cancelled, unmounted mid-run), text must not be stuck invisible. Final state is the safe state.
5. **Keep it short and non-repeating.** No auto-loop. Anything auto-playing longer than 5s needs a pause control (WCAG 2.2.2) — another reason to cap total duration.
6. **Respect text scaling.** Don't hard-code pixel translate distances that break with Dynamic Type / browser zoom; derive the offset from the character's measured or font-relative height.
7. **Grapheme-safe splitting.** Split with `Intl.Segmenter` (granularity `grapheme`) when available, falling back to `Array.from`, so emoji like 👩🏽‍🌾 and accented letters aren't torn apart.

When you finish, briefly tell the user which of these you applied — they care about this.

## Workflow

1. **Detect the platform.** `react-native` in package.json → React Native. `next` / `react-dom` → web. Both are covered below.
2. **Find where it goes.** If the user hasn't said which text to animate and when it should play (on mount, on scroll into view, when new content arrives, on a Replay button), ask — don't guess at placement in someone's screen.
3. **Add the component** from the matching reference file (or `references/typing.md` for typing) into the project's existing components location, matching the project's language (TS vs JS), import style, and formatting. Don't add dependencies — both versions use only built-ins (RN `Animated`, plain CSS transitions).
   - React Native → read `references/react-native.md`
   - React / Next.js web → read `references/web-react.md`
4. **Match the project's typography.** Pass the font family, size, weight, color, and line height the surrounding text already uses, rather than the demo's 72px Google Sans.
5. **Wire up the trigger** the user asked for. For content that arrives later (e.g., a new comment appearing), mount the component when the content appears, or flip its `play` prop.
6. **Verify** if you can run the app: the text should end fully visible with no clipped descenders, screen reader should read it as one string, and with reduced motion on it should appear instantly.

## Props both implementations share

```
text: string            // supports "\n" for explicit line breaks
splitBy: "characters" | "words"     = "characters"
staggerFrom: "first" | "last" | "center" | "random" = "first"
staggerMs: number       = 25
durationMs: number      = 600
reverse: boolean        = false
play: boolean           = true      // false → hidden-ready state; true → animate in
delayMs: number         = 0
onComplete?: () => void
```

Plus a style prop for the text and a heading/role option. Replay = flip `play` false → true (or change a `key`).
