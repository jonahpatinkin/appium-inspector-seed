# inspector_seed

`inspector_seed` turns a recorded Appium Inspector session into a seed
navigation map for an Android app, by replaying that recording against a
real device.

## What problem this solves

Walking an app by hand in Appium Inspector and copying out the generated
Python code is a fast way to capture one real path through an app. But
that generated code is just a flat list of driver calls — it has no
notion of "screens," "elements," or "transitions." `inspector_seed` reads
that flat recording, replays it on a real device, and while doing so
builds a structured map: which screens exist, what clickable elements
each one has, and which element-tap leads to which other screen.

That map can then seed a broader, automated exploration of the rest of
the app, instead of starting from nothing.

## The three stages

```
recording.py (Appium Inspector's generated code)
        |
        v
   [1] parser.parse()        -- turns raw text into structured steps
        |
        v
   [2] replay.build_seed_nav_map()  -- drives a real device through those steps
        |
        v
   [3] cli.py                -- wires 1+2 together, writes the result to disk
```

### 1. Parsing (`parser.py`)

Appium Inspector's Python code generator has a specific, narrow shape.
For each element the human interacted with, it emits an assignment line
followed by an action line, sometimes with other (ignorable) lines in
between:

```python
el7 = driver.find_element(by=AppiumBy.ID, value="some_value")
...
el7.click()
```

The same variable is reused if the human tapped the same element more
than once; it is never reassigned. The parser's job is to read every
line of the recording and turn it into one of three outcomes:

- **An assignment** — recorded internally (which strategy + value a
  variable refers to), produces no step by itself.
- **A recognized action** (`.click()` / `.send_keys(...)`, or
  `driver.back()`) — resolved against whatever its variable was last
  assigned to, and turned into a structured step (`tap`, `fill`, or
  `back`) with a locator strategy and value.
- **An unrecognized line** — anything that doesn't match one of the
  shapes above becomes an `UNPARSEABLE` entry instead of being silently
  dropped. If a variable's action line is reached but that variable was
  never actually assigned (the code generator sometimes does this for
  locator strategies it has no code-generation support for), that line
  is unparseable too — never guessed at.

If the recording has ANY unparseable lines, replay never starts at all —
the whole recording is rejected up front, before touching a device, so a
bad recording never burns device time. The caller gets back every
offending line and its line number, to fix and re-record.

Supported locator strategies cover the realistic Android surface a
device-based app can produce (element ID, accessibility ID, CSS
selector, XPath, class name, name, and a few Android-specific UI
Automator / data-matcher / view-tag strategies) — iOS-only strategies
are deliberately out of scope.

### 2. Replay (`replay.py`)

This is the part that actually drives the device. For each parsed step,
in order:

1. **Pace like a human.** A short, randomized pause happens before every
   step. A real human recording a walkthrough naturally pauses between
   actions; replaying every step back-to-back as fast as code allows
   races ahead of screens that are still loading.
2. **Detect the current screen.** Figures out whether the driver is
   currently looking at a native screen or a WebView-hosted one, and
   enumerates that screen's clickable elements (building up the map's
   per-screen element list as it goes — not just the one element each
   step targets).
3. **Decide native vs. WebView for the upcoming action.** This decision
   is made per-step, based on THIS step's own locator strategy — not on
   whether the current screen generally has a WebView available. A
   single logical screen can legitimately offer both a native layer and
   a WebView at the same time (a human can discover a WebView is present
   and still tap a native element on purpose); deciding per-screen
   instead of per-step would route some actions through the wrong driver
   layer entirely.
4. **Perform the action, with retry.** Even with the pacing pause, some
   screens can take noticeably longer to fully render before their
   target element becomes findable — this varies at runtime and isn't
   predictable up front. The actual tap/fill call is retried for a
   bounded window if the element isn't found yet, rather than failing on
   the very first miss.
5. **Record the transition.** After the action, the screen is checked
   again — if it changed, a transition is recorded on the screen the
   step started from (which element, what action, which screen it led
   to).
6. **Dump the screen on every step.** Regardless of success or failure,
   the live screen's current UI structure is written to disk, labeled
   with that step's number and outcome. If a step genuinely fails, that
   dump is also copied to one well-known, stable path — so a human can
   see exactly what the device actually showed at the moment of failure,
   without re-running anything.

The whole replay (install → launch → replay every step → uninstall) is
wrapped in continuous activity-log capture, which is correlated against
the actions actually performed once replay finishes.

### 3. CLI (`cli.py`)

Thin wiring only — no parsing or replay logic of its own:

```
python -m inspector_seed.cli --apk <path-to-apk> --recording <path-to-recording.py> [--serial <device-id>] [--out <path>]
```

- Reads the recording file and parses it.
- If anything is unparseable, prints every offending line and exits
  non-zero without touching a device.
- Otherwise replays it against the given device, writes the resulting
  navigation map (and the correlated activity log) to disk, and prints a
  short summary: how many screens, transitions, and elements were found.

## Why a step can still fail

Not every real-world mismatch is a timing problem. A recording captures
one specific run through the app at one specific moment — if the
account, app, or backend state has since moved past what the recording
assumed (a one-time setup screen that only shows once, a value that was
only ever valid for that one session), the step the recording expects
simply won't exist anymore, no matter how long you wait for it. When
that happens, the per-step screen dump described above is the first
place to look: it shows exactly what the device had on screen instead of
what the recording expected, which is usually enough to tell whether
that one step just needs to be re-recorded.
