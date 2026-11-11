# How to run inspector_seed from OpenCode

`inspector-seed-runner.md` is an agent playbook, not a script you run
yourself — you run it by giving OpenCode a plain-English prompt. OpenCode
(acting as that agent) then reads `inspector-seed.md` +
`inspector-seed-fix-history.md`, confirms device/server/interpreter, and runs
the actual `python -m inspector_seed.cli` command on your behalf.

## Example prompt

```
Read inspector-seed-runner.md and use it to run the Appium recording
"recordings/recording.py" through inspector_seed
against my connected device. APK is <path-to-apk>.
Tell me the exit code and, if it fails, which step and dump to look at.
```

Shorter, if the device is already obvious/known to the project but the APK
still needs to be pointed at explicitly:

```
Using inspector-seed-runner.md, replay
"recordings/recording.py" against
<path-to-apk>
and report what happened.
```

If you want it to pick the device itself:

```
Follow inspector-seed-runner.md's playbook to run
recording.py. Check `adb devices` and the Appium
server yourself first — don't ask me, just confirm them and proceed.
```

## What OpenCode will actually do with that prompt

Per `inspector-seed-runner.md`'s own steps, it will:

1. Read `inspector-seed.md` and `inspector-seed-fix-history.md` in full first.
2. Confirm: the recording file looks like real Inspector-generated code,
   `adb devices` shows the serial as `device`, and
   `curl http://127.0.0.1:4723/status` responds — asking you for whichever of
   serial/APK path it can't find on its own.
3. Find a Python interpreter with Appium installed (checks
   `python3 -c "import appium"`, falls back to a project `.venv` if needed).
4. Run:
   ```bash
   adb -s <serial> uninstall <package-id> 2>/dev/null || true
   PYTHONPATH=src <python> -m inspector_seed.cli \
     --apk <apk-path> \
     --recording "recordings/recording.py" \
     --serial <serial>
   ```
5. Report back: exit code; on success, the screens/transitions/elements
   summary verbatim; on failure, the exact step number, exception type, the
   matching known failure class from the fix history (if any), and which
   `artifacts/inspector_seed_steps/step_NNN.xml` /
   `artifacts/inspector_seed_failed_step_page_source.xml` dump to check.
6. Confirm the device was left clean (`<package-id>` no longer installed).

## What it will refuse to do, even if asked

- Patch `inspector_seed` source to "fix" an undiagnosed failure.
- Re-record the `.py` file itself (e.g. the login/password steps) — it'll
  name the exact failing step and ask you to re-record it in Appium
  Inspector instead.
- Set up the emulator, Appium server, or build the APK for you — that's
  flagged as out of scope, not guessed at.

## If you want it as a dedicated OpenCode agent instead of a one-off prompt

Copy `inspector-seed-runner.md` into OpenCode's agent directory (project
`.opencode/agent/inspector-seed-runner.md` or global
`~/.config/opencode/agent/`), then either switch to it directly in OpenCode
or invoke it as a subagent by name, passing the same prompt shown above.
