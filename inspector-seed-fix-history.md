# inspector_seed — fix history and how we got it working

This is the story of `inspector_seed` in the order the problems were
actually found and fixed, each one discovered by running the real tool
against a real device. See `src/inspector_seed/README.md` for how the
finished tool works; this document is about how it got there.

## 1. Locator strategies kept being wrong (css → xpath → everything else)

**First finding:** the parser only recognized `css selector` as a
WebView locator strategy. A real recording proved Appium Inspector's own
code generator never actually emits `css selector` at all, in any
language — for WebView elements it emits **XPath** instead. Confirmed by
reading Inspector's own generator source directly, not guessed at. Fixed
by adding `xpath` as a supported strategy end to end (the shared driver
layer, the parser, and replay).

**Second finding, immediately after:** two more real recordings hit two
more unsupported-strategy gaps in a row — `class name` and
`-android uiautomator` (a different Android-native locator style),
sometimes both in the SAME recording. Rather than patch one strategy at
a time forever, the fix generalized to support every Android-relevant
locator strategy Inspector's generator can actually produce (confirmed
against Inspector's own source, not guessed), explicitly excluding
iOS-only ones since this project targets Android only.

**A real bug surfaced alongside this:** one of the new strategies
(`-android uiautomator`) can carry a selector value containing literal
backslash-escaped quotes. The parser's original quote-matching logic
stopped at the FIRST escaped quote it saw, silently truncating the
captured value. Fixed by making the quote-matching logic treat an
escaped quote as a single unit instead of a string terminator, and by
un-escaping the captured value before it's used (a raw backslash left in
a locator string is a different, wrong string to a live driver).

## 2. The parser's whole grammar was wrong (not just the strategies)

**Finding:** running the parser against a real recording returned
**zero** recognized steps and **zero** rejected lines — every single
real action in the file silently vanished. The parser had been built to
recognize a single-line pattern
(`driver.find_element(strategy, "value").click()`). A real recording
proved Appium Inspector never actually generates that shape, for ANY
locator strategy. The real shape is always two separate statements — an
assignment, then an action, sometimes much later:

```python
el7 = driver.find_element(by=..., value="...")
...
el7.click()
```

The SAME variable gets reused (never reassigned) for a repeat action on
an already-resolved locator. The parser was rewritten from scratch
around this real two-statement shape: track every assignment in a
lookup table as the file is scanned, then resolve each action line
against that table. A variable referenced on an action line that was
never actually assigned anywhere in the file (which happens when
Inspector has no code generator support for a given locator strategy —
it prints a comment instead of an assignment, but the action line
referencing that variable still gets generated) now becomes a clearly
rejected line instead of silently vanishing.

## 3. A screen can be "native" and "WebView" at the same time

**Finding, from a real device crash:** the replay engine decided whether
to force the driver back to native-app mode based on whether the
CURRENT SCREEN generally had a WebView available anywhere on it — not on
whether the step actually about to happen needed the WebView. A real
recording proved a single logical screen can legitimately offer both a
native element and a WebView at the same time (a human can discover a
WebView is present, and still deliberately tap a native element instead).
When that happened, the old logic skipped forcing native mode, and a
native-only action got routed through the WebView's own browser engine,
which flatly rejects any locator strategy it doesn't understand.

**Fix:** the native-vs-WebView decision was changed to depend on the
step's OWN locator strategy, not on screen-level WebView availability.
Only one locator strategy is ever used for a genuine WebView-DOM
selector; every other strategy (including a plain "go back" action)
always forces native mode first, regardless of what else the screen
happens to offer.

## 4. Real devices don't load instantly

**Finding, measured directly on a real device:** right after a tap that
triggers a screen change into a WebView-hosted page, the new page's
content does not attach to the driver's view of the screen right away —
confirmed by polling immediately after the tap and watching the result
change about a second later. Worse, a SPECIFIC element on that same page
was measured taking as long as **7 seconds** to become reliably
findable, varying between runs.

**Two separate fixes, because one wasn't enough:**

- **A short pause before every step.** A human naturally pauses between
  actions while recording a walkthrough; replaying every step back to
  back as fast as code allows races ahead of what's actually loaded. A
  brief, randomized pause before each step — the exact same mechanism
  already used elsewhere for pacing an automated walkthrough at
  human speed — covers the common case.
- **A retry loop around the action itself.** The flat pause alone wasn't
  enough for the slow 7-second case measured live. The actual tap/fill
  call now retries for a bounded window (several seconds) if the target
  element isn't found yet, instead of giving up on the very first
  attempt. Any OTHER kind of failure (a genuinely wrong locator, a
  driver-level problem) is never retried — only "not found yet" is.

## 5. Not every failure is fixable by waiting longer

**Finding:** even with the pause and the retry, some steps still
genuinely fail — not because the device is slow, but because the exact
element the recording expects isn't there anymore, no matter how long
you wait. This happens when a step's target was tied to one specific,
one-time state (a setup screen that only shows once, a value or
identifier that was only ever valid for one particular earlier session)
and that state has since moved on.

**What this needed instead of a code fix:** visibility. A one-off,
throwaway script had to be written by hand, more than once, just to
connect to the device and dump its current screen to see what was
actually there. That manual step is now automatic: the live screen's
structure is written to disk after EVERY step (success or failure), each
one labeled with its step number and outcome, plus the failing step's
dump is also copied to one well-known, stable file path. Instead of
guessing from a crash, the actual, current state of the device is
sitting on disk, ready to read.

## 6. Status

All of the above are implemented, tested (including against the real
device, not just with mocked unit tests), and verified. What's still
open, not yet implemented: deciding, automatically, that a step whose
target genuinely isn't there anymore was an optional, skippable step
(rather than a real failure) by checking whether a LATER step's target
is already on screen — so the replay can continue past a screen that
legitimately no longer appears, instead of stopping there every time.
