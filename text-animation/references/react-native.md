# React Native implementation

Uses only `react-native` built-ins (`Animated`, `Easing`, `AccessibilityInfo`). Animations run on the native driver (`transform` only), so they stay smooth while JS is busy.

How it maps from the web version:
- CSS `overflow: hidden` clip box → `View` with `overflow: 'hidden'`
- `translateY(120%)` → RN transforms don't accept percentages, so the offset is computed from the font size × line-height ratio × the user's font scale (Dynamic Type / Android font size), which keeps it correct at large text sizes
- Per-element `setTimeout`s → one `Animated.parallel` of `Animated.timing`s with per-unit `delay`, so a single `.stop()` cancels everything
- Word wrapping → each line is a `flexDirection: 'row', flexWrap: 'wrap'` row of word groups; each word group never breaks internally

Save as `SplitTextReveal.tsx` next to the screen components (or the project's components folder). Drop the type annotations if the project is JS.

```tsx
import React, { useEffect, useMemo, useState } from 'react';
import {
  AccessibilityInfo,
  AccessibilityRole,
  Animated,
  Easing,
  StyleProp,
  StyleSheet,
  Text,
  TextStyle,
  useWindowDimensions,
  View,
  ViewStyle,
} from 'react-native';

type StaggerFrom = 'first' | 'last' | 'center' | 'random';

type Props = {
  text: string; // "\n" = explicit line break
  splitBy?: 'characters' | 'words';
  staggerFrom?: StaggerFrom;
  staggerMs?: number;
  durationMs?: number;
  reverse?: boolean; // drop in from above instead of rising from below
  play?: boolean; // false = hidden and ready; true = animate in
  delayMs?: number;
  onComplete?: () => void;
  textStyle?: StyleProp<TextStyle>;
  style?: StyleProp<ViewStyle>;
  accessibilityRole?: AccessibilityRole; // e.g. 'header' for headings
  respectReducedMotion?: boolean; // false for screen recordings / demos
};

const EASING = Easing.bezier(0.16, 1, 0.3, 1);

// Grapheme-safe split so emoji and accented letters stay whole.
function splitGraphemes(s: string): string[] {
  const Segmenter = (Intl as any)?.Segmenter;
  if (Segmenter) {
    return Array.from(
      new Segmenter(undefined, { granularity: 'grapheme' }).segment(s),
      (seg: any) => seg.segment,
    );
  }
  return Array.from(s);
}

// rank[i] = position of unit i in the reveal sequence.
function staggerRanks(count: number, from: StaggerFrom): number[] {
  const order = [...Array(count).keys()];
  if (from === 'last') order.reverse();
  else if (from === 'center') {
    const c = Math.floor(count / 2);
    order.sort((a, b) => Math.abs(a - c) - Math.abs(b - c));
  } else if (from === 'random') order.sort(() => Math.random() - 0.5);
  const ranks = new Array(count);
  order.forEach((unitIndex, rank) => (ranks[unitIndex] = rank));
  return ranks;
}

export default function SplitTextReveal({
  text,
  splitBy = 'characters',
  staggerFrom = 'first',
  staggerMs = 25,
  durationMs = 600,
  reverse = false,
  play = true,
  delayMs = 0,
  onComplete,
  textStyle,
  style,
  accessibilityRole = 'text',
  respectReducedMotion = true,
}: Props) {
  const { fontScale } = useWindowDimensions();
  const [reduceMotion, setReduceMotion] = useState<boolean | null>(null);

  useEffect(() => {
    let mounted = true;
    if (!respectReducedMotion) {
      setReduceMotion(false);
      return;
    }
    AccessibilityInfo.isReduceMotionEnabled().then(v => mounted && setReduceMotion(v));
    const sub = AccessibilityInfo.addEventListener('reduceMotionChanged', setReduceMotion);
    return () => {
      mounted = false;
      sub.remove();
    };
  }, [respectReducedMotion]);

  // lines -> words -> units, each unit tagged with a global index
  const { lines, count } = useMemo(() => {
    let i = 0;
    const built = text.split('\n').map(line =>
      line.split(/ +/).filter(Boolean).map(word =>
        (splitBy === 'words' ? [word] : splitGraphemes(word)).map(unit => ({ unit, index: i++ })),
      ),
    );
    return { lines: built, count: i };
  }, [text, splitBy]);

  const ranks = useMemo(() => staggerRanks(count, staggerFrom), [count, staggerFrom]);
  const anims = useMemo(
    () => Array.from({ length: count }, () => new Animated.Value(0)),
    // new values whenever the text changes
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [text, splitBy],
  );

  useEffect(() => {
    if (!play) {
      anims.forEach(a => a.setValue(0));
      return;
    }
    if (reduceMotion === null) return; // wait for the setting before moving anything
    if (reduceMotion) {
      anims.forEach(a => a.setValue(1)); // final state, no movement
      onComplete?.();
      return;
    }
    anims.forEach(a => a.setValue(0));
    const run = Animated.parallel(
      anims.map((a, i) =>
        Animated.timing(a, {
          toValue: 1,
          duration: durationMs,
          delay: delayMs + ranks[i] * staggerMs,
          easing: EASING,
          useNativeDriver: true,
        }),
      ),
    );
    run.start(({ finished }) => finished && onComplete?.());
    return () => {
      run.stop();
      anims.forEach(a => a.setValue(1)); // never leave text stuck hidden
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [play, reduceMotion, anims, ranks, staggerMs, durationMs, delayMs]);

  const flat = StyleSheet.flatten(textStyle) ?? {};
  const fontSize = (flat.fontSize ?? 14) * fontScale;
  const lineHeight = flat.lineHeight ? flat.lineHeight * fontScale : fontSize * 1.2;
  const offset = lineHeight * 1.2 * (reverse ? -1 : 1);
  const clip = {
    paddingTop: fontSize * 0.15,
    paddingBottom: fontSize * 0.25,
    marginTop: -fontSize * 0.15,
    marginBottom: -fontSize * 0.25,
  };

  return (
    <View
      style={style}
      accessible
      accessibilityRole={accessibilityRole}
      accessibilityLabel={text.replace(/\n/g, ' ')}
    >
      {/* Split pieces are visual only; the wrapper's label is what assistive tech reads. */}
      <View importantForAccessibility="no-hide-descendants" accessibilityElementsHidden>
        {lines.map((words, li) => (
          <View key={li} style={styles.line}>
            {words.map((units, wi) => (
              <View key={wi} style={styles.word}>
                {units.map(({ unit, index }) => (
                  <View key={index} style={[styles.clip, clip]}>
                    <Animated.Text
                      style={[
                        textStyle,
                        {
                          transform: [
                            {
                              translateY: anims[index].interpolate({
                                inputRange: [0, 1],
                                outputRange: [offset, 0],
                              }),
                            },
                          ],
                        },
                      ]}
                    >
                      {unit}
                    </Animated.Text>
                  </View>
                ))}
                {wi < words.length - 1 && <Text style={textStyle}> </Text>}
              </View>
            ))}
          </View>
        ))}
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  line: { flexDirection: 'row', flexWrap: 'wrap' },
  word: { flexDirection: 'row' },
  clip: { overflow: 'hidden' },
});
```

## Usage

```tsx
// Headline on mount
<SplitTextReveal
  text={'The needs of the many\noutweigh the needs of the\nfew, or the one.'}
  textStyle={{ fontSize: 40, lineHeight: 38, color: '#1a1a1a' }}
  accessibilityRole="header"
/>

// Content that arrives later (e.g. a new reply): mount it when it appears
{reply && <SplitTextReveal text={reply.text} splitBy="words" textStyle={styles.replyText} />}

// Replay: change the key
<SplitTextReveal key={replayCount} text={title} textStyle={styles.title} />
```

## Notes
- For body text (comments, captions) prefer `splitBy="words"` — per-letter staggering on a 100-character comment takes ~3s and feels sluggish.
- Each character is its own `Text`, so kerning between letters is lost. That's acceptable for display type; for small body text, words look noticeably better.
- If the project's text style uses a custom font via `fontFamily`, pass it in `textStyle` so the spaces between words match too.
