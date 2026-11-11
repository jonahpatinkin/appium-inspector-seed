# appium-inspector-seed

`inspector_seed` turns a recorded Appium Inspector session into a seed
navigation map for an Android app, by replaying that recording against a
real device. See [`inspector-seed.md`](inspector-seed.md) for the full
design (parsing, replay, CLI).

> **Status:** this repository currently holds the design docs and the
> OpenCode agent playbook only. The `inspector_seed` Python package
> (`src/inspector_seed/{parser,replay,cli}.py`) referenced throughout
> these docs has not been committed yet and will land in a follow-up
> commit.

## Docs in this repo

- [`inspector-seed.md`](inspector-seed.md) — what the tool does and how
  the three stages (parse → replay → CLI) fit together.
- [`inspector-seed-fix-history.md`](inspector-seed-fix-history.md) — the
  real bugs hit while building it, in the order they were found and
  fixed.
- [`inspector-seed-runner.md`](inspector-seed-runner.md) — the OpenCode
  agent playbook that drives a run end to end.
- [`inspector-seed-how-to-run.md`](inspector-seed-how-to-run.md) — how to
  invoke that playbook from OpenCode, with example prompts.

## Quick start (once the source lands)

```bash
python -m inspector_seed.cli \
  --apk <path-to-apk> \
  --recording <path-to-recording.py> \
  [--serial <device-id>] [--out <path>]
```

Or, from OpenCode:

```
Read inspector-seed-runner.md and use it to run the Appium recording
"recordings/recording.py" through inspector_seed
against my connected device. APK is <path-to-apk>.
```
