# React / Next.js web implementation

Pure CSS transitions — one state flip reveals everything, and each unit's stagger is a `transition-delay`, so there are no per-letter timers to clean up. No dependencies; works with Tailwind or plain CSS.

Improvements over the original v0 component, and why:
- Full text is in a visually hidden span and the split letters are `aria-hidden`, so screen readers read one sentence instead of letter by letter.
- Reduced motion is handled in CSS (`prefers-reduced-motion`), so it applies even before JavaScript loads.
- A `<noscript>` override means text is never stuck hidden if JS fails.
- Real space characters between words, so line wrapping and spacing match the font.
- Renders as any element (`as="h1"`), keeping heading semantics.

Save as `components/split-text-reveal.tsx` (match the project's file naming). Keep `"use client"` for the Next.js App Router.

```tsx
"use client"

import { useEffect, useMemo, useState, type CSSProperties, type ElementType } from "react"

type StaggerFrom = "first" | "last" | "center" | "random"

type Props = {
  text: string // "\n" = explicit line break
  as?: ElementType // "h1", "h2", "p"…
  splitBy?: "characters" | "words"
  staggerFrom?: StaggerFrom
  staggerMs?: number
  durationMs?: number
  reverse?: boolean
  play?: boolean // false = hidden and ready; true = animate in
  delayMs?: number
  onComplete?: () => void
  className?: string
  style?: CSSProperties
  respectReducedMotion?: boolean // false for screen recordings / demos
}

const EASING = "cubic-bezier(0.16, 1, 0.3, 1)"

function splitGraphemes(s: string): string[] {
  const Segmenter = (Intl as any)?.Segmenter
  if (Segmenter) {
    return Array.from(new Segmenter(undefined, { granularity: "grapheme" }).segment(s), (seg: any) => seg.segment)
  }
  return Array.from(s)
}

function staggerRanks(count: number, from: StaggerFrom): number[] {
  const order = [...Array(count).keys()]
  if (from === "last") order.reverse()
  else if (from === "center") {
    const c = Math.floor(count / 2)
    order.sort((a, b) => Math.abs(a - c) - Math.abs(b - c))
  } else if (from === "random") order.sort(() => Math.random() - 0.5)
  const ranks = new Array(count)
  order.forEach((unitIndex, rank) => (ranks[unitIndex] = rank))
  return ranks
}

const CSS = `
.str-clip{position:relative;display:inline-block;overflow:hidden;vertical-align:top;
  padding-top:.15em;padding-bottom:.25em;margin-top:-.15em;margin-bottom:-.25em}
.str-unit{display:inline-block;will-change:transform}
.str-word{display:inline-block;white-space:nowrap}
.str-line{display:block}
.str-sr{position:absolute;width:1px;height:1px;padding:0;margin:-1px;overflow:hidden;clip:rect(0,0,0,0);white-space:nowrap;border:0}
`
const REDUCED_MOTION_CSS = `@media (prefers-reduced-motion: reduce){.str-unit{transform:none!important;transition:none!important}}`

export function SplitTextReveal({
  text,
  as: Tag = "p",
  splitBy = "characters",
  staggerFrom = "first",
  staggerMs = 25,
  durationMs = 600,
  reverse = false,
  play = true,
  delayMs = 0,
  onComplete,
  className,
  style,
  respectReducedMotion = true,
}: Props) {
  const [revealed, setRevealed] = useState(false)

  const { lines, count } = useMemo(() => {
    let i = 0
    const built = text.split("\n").map((line) =>
      line
        .split(/ +/)
        .filter(Boolean)
        .map((word) => (splitBy === "words" ? [word] : splitGraphemes(word)).map((unit) => ({ unit, index: i++ }))),
    )
    return { lines: built, count: i }
  }, [text, splitBy])

  const ranks = useMemo(() => staggerRanks(count, staggerFrom), [count, staggerFrom])

  useEffect(() => {
    setRevealed(false)
    if (!play) return
    // Two frames so the hidden state paints before the transition starts (needed for replay).
    let r2 = 0
    const r1 = requestAnimationFrame(() => {
      r2 = requestAnimationFrame(() => setRevealed(true))
    })
    const reduce = respectReducedMotion && window.matchMedia("(prefers-reduced-motion: reduce)").matches
    const total = reduce ? 0 : delayMs + Math.max(0, count - 1) * staggerMs + durationMs
    const done = onComplete ? window.setTimeout(onComplete, total) : undefined
    return () => {
      cancelAnimationFrame(r1)
      cancelAnimationFrame(r2)
      window.clearTimeout(done)
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [play, text, splitBy, staggerFrom, staggerMs, durationMs, delayMs])

  const hidden = `translateY(${reverse ? "-120%" : "120%"})`

  return (
    <Tag className={className} style={style}>
      <style>{respectReducedMotion ? CSS + REDUCED_MOTION_CSS : CSS}</style>
      <noscript>
        <style>{`.str-unit{transform:none!important}`}</style>
      </noscript>
      <span className="str-sr">{text.replace(/\n/g, " ")}</span>
      <span aria-hidden="true">
        {lines.map((words, li) => (
          <span key={li} className="str-line">
            {words.map((units, wi) => (
              <span key={wi}>
                <span className="str-word">
                  {units.map(({ unit, index }) => (
                    <span key={index} className="str-clip">
                      <span
                        className="str-unit"
                        style={{
                          transform: revealed ? "translateY(0)" : hidden,
                          transition: revealed
                            ? `transform ${durationMs}ms ${EASING} ${delayMs + ranks[index] * staggerMs}ms`
                            : "none",
                        }}
                      >
                        {unit}
                      </span>
                    </span>
                  ))}
                </span>
                {wi < words.length - 1 && " "}
              </span>
            ))}
          </span>
        ))}
      </span>
    </Tag>
  )
}
```

## Usage

```tsx
<SplitTextReveal
  as="h1"
  text={"The needs of the many\noutweigh the needs of the\nfew, or the one."}
  className="text-[72px] leading-[0.95] tracking-[-0.01em] text-neutral-900"
/>

// Replay button
const [run, setRun] = useState(0)
<button onClick={() => setRun((n) => n + 1)}>Replay</button>
<SplitTextReveal key={run} as="h2" text={headline} />

// Reveal on scroll: drive `play` from an IntersectionObserver
<SplitTextReveal text={headline} play={inView} />
```

## Notes
- The original v0 demo also scaled the whole lockup down to fit narrow screens with `transform: scale()`. Don't carry that over by default — scaled text ignores browser zoom and can get very small. Use responsive font sizes (`clamp()` or breakpoint classes) instead, and only add fit-to-width scaling if the user explicitly wants a fixed lockup.
- For body copy, use `splitBy="words"`.
