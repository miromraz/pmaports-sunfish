# Rubber Ducky — load scripts from files + bundled examples

**Date:** 2026-05-22
**Component:** `community/rubberducky-pm/`
**Status:** approved design

## Goal

Two user-facing additions to the Rubber Ducky app:

1. **File-picker import** — bring a DuckyScript `.txt` file from the phone's
   filesystem into the payload library.
2. **Auto-seeded examples** — ship a curated set of real, harmless DuckyScript
   payloads and copy them into the payload library on first launch, so the
   user has runnable scripts immediately (for testing the run path on real
   hardware).

## Background

Today payloads live as `.txt` files in `AppDataLocation/payloads/`, created
only through the in-app editor. The `examples/` directory holds one file
(`hello-world.txt`) but the APKBUILD `package()` step never installs it, so it
never reaches the phone. There is no way to bring in a script from outside the
app.

The app is a `QGuiApplication` (Qt Quick, no QtWidgets), so a Python
`QFileDialog` is unavailable — the file picker must be QML's `FileDialog`
from `QtQuick.Dialogs`.

## Feature 1 — file-picker import

### UI

`qml/main.qml`, Payloads list page (`listPage`): add an **Import** action to
the page's `actions` array.

```qml
Kirigami.Action {
    text: "Import"
    icon.name: "document-import"
    enabled: !Backend.running
    onTriggered: importDialog.open()
}
```

Add a `FileDialog` (sibling of the existing `OverlaySheet`s):

```qml
FileDialog {
    id: importDialog
    title: "Import DuckyScript"
    currentFolder: Backend.examplesDir
    nameFilters: ["DuckyScript (*.txt)", "All files (*)"]
    onAccepted: {
        var name = Backend.importPayload(selectedFile);
        if (name)
            root.showPassiveNotification("Imported as " + name);
        else
            root.showPassiveNotification("Could not import file");
    }
}
```

Requires `import QtQuick.Dialogs` at the top of `main.qml`.

### Backend

`rubberducky_pm/app.py`, class `Backend`:

- **`examplesDir`** — `@Property(QUrl, constant=True)`. Returns the bundled
  examples directory as a `QUrl` (see "Locating the examples directory"
  below), used as the `FileDialog`'s starting folder so the bundled examples
  are one tap away. Falls back to a standard writable location if the
  examples directory is absent, so the dialog always opens somewhere valid.

- **`importPayload(QUrl) -> str`** — `@Slot(QUrl, result=str)`:
  1. `path = url.toLocalFile()`; read it as UTF-8.
  2. On `OSError` / `UnicodeDecodeError` → return `""`.
  3. If the content is empty or whitespace-only → return `""`.
  4. Base name = filename stem (`my-script.txt` → `my-script`); if empty,
     use `"imported"`.
  5. `name = unique_name(base, self._names)` — collision → `-2`, `-3`, …
     so an import never overwrites an existing payload.
  6. Write the content to `_path_for(name)`. On `OSError` → return `""`.
  7. `_rescan()`, return `name`.

Import does not compile the script — the existing Validate/Run paths already
surface compile errors. The Import action does not touch USB; the
`enabled: !Backend.running` guard is for UI consistency only.

## Feature 2 — auto-seeded examples

### Seeding logic

`rubberducky_pm/app.py`, in `Backend.__init__`, after the payload directory
is created and before `_rescan()`:

- Read a `QSettings` flag `examplesSeeded` (bool, default `False`).
- If not set, call `_seed_examples()`, then set the flag.

`_seed_examples()`:
- List `*.txt` in the examples directory.
- For each, if `_path_for(stem)` does **not** already exist, copy the file
  into the payload directory (`shutil.copyfile`).
- Best-effort: wrap in `try/except OSError`, write a one-line diagnostic to
  `stderr` on failure. Seeding must never crash app launch.

Behaviour this gives:
- Examples appear in the payload list on first launch, ready to run.
- The flag makes seeding run **once** — deleting an example does not
  resurrect it on the next launch.
- A name collision with a user's own payload is skipped — seeding never
  overwrites user content.
- Existing installs upgrading to this release get the examples seeded once.

### Locating the examples directory

Add `_examples_dir()`, mirroring the existing `_resource_dir()`:

1. `RUBBERDUCKY_EXAMPLES_DIR` environment override (for development).
2. Installed path: `/usr/share/rubberducky-pm/examples`.
3. Source-tree fallback: `<repo>/examples` relative to the module.

Returns the first existing directory, or `None`.

### Bundled example scripts

All curated/adapted from public sources, every one vetted harmless and
confirmed to compile under the supported DuckyScript 1.0 subset and US
keymap. Each carries a `REM` header.

| File | Source | Behaviour (all harmless) |
|-|-|-|
| `hello-world.txt` | existing | Opens Run dialog, types a greeting — runs nothing |
| `rickroll.txt` | classic | Opens the browser to the YouTube video — one tab |
| `matrix-wake-up.txt` | Hak5 / UberGuidoZ, attributed | Opens cmd, types the "Wake up, Neo…" scene, beeps, closes the window |
| `windows-notepad.txt` | Hak5 docs classic | Opens Notepad, types a short multi-line note |
| `linux-terminal.txt` | adapted for KDE | KRunner → Konsole → `echo` a greeting; runs on the phone's own KDE host — handy for testing |

`hello-world.txt` is kept verbatim. The four new files:

**`rickroll.txt`**
```
REM Title: Rickroll
REM Description: Opens the classic music video in the default browser.
REM Harmless: opens one browser tab, nothing else.
REM Target: Windows (Run dialog)
DELAY 1000
GUI r
DELAY 500
STRING https://www.youtube.com/watch?v=dQw4w9WgXcQ
DELAY 300
ENTER
```

**`windows-notepad.txt`**
```
REM Title: Notepad Note
REM Description: Opens Notepad and types a short multi-line note.
REM Harmless: types text only.
REM Target: Windows
DELAY 1000
GUI r
DELAY 500
STRING notepad
ENTER
DELAY 1500
STRINGLN Hello from a postmarketOS phone!
STRINGLN
STRINGLN This note was typed over USB HID by Rubber Ducky.
STRING Authorised testing only.
```

**`linux-terminal.txt`**
```
REM Title: Linux Terminal Greeting
REM Description: Opens KRunner, launches Konsole, echoes a greeting.
REM Harmless: runs only 'echo'. Runs on the phone's own KDE host.
REM Target: Linux / KDE Plasma
DELAY 1000
ALT F2
DELAY 600
STRING konsole
ENTER
DELAY 1500
STRINGLN echo "Hello from a postmarketOS phone over USB HID"
```

**`matrix-wake-up.txt`** — adapted from the Hak5 payload
`payloads/library/prank/The_Matrix-Wake_Up/payload.txt` by UberGuidoZ. The
`ATTACKMODE HID STORAGE` line (DuckyScript 3.0, unsupported) is removed; the
original `REM` attribution header is kept. The script opens cmd, sets a green
console colour, types the Matrix dialogue character-by-character, plays a
beep via PowerShell, and closes the window with `ALT F4`. Authored faithfully
to the original at implementation time, verified to compile.

## Shared / supporting changes

### New module `rubberducky_pm/library.py`

A Qt-free module holding the pure helper:

- **`unique_name(base, existing) -> str`** — returns `base` if not in
  `existing`, otherwise the first free `base-2`, `base-3`, … .

Rationale for a separate module: `app.py` imports `PyQt6` at module scope,
and the abuild `check()` environment has no `py3-qt6` (it is a runtime
`depends`, not a `makedepends`). Keeping `unique_name` Qt-free lets `check()`
unit-test it. `Backend` imports `unique_name` from this module.

### APKBUILD

- `package()`: create `$share/examples` and
  `install -m644 "$builddir"/examples/*.txt "$share/examples/"`.
- `check()`: add `python3 "$builddir"/tests/test_library.py` and
  `python3 "$builddir"/tests/test_examples.py`.
- Bump `pkgrel` 3 → 4.
- Regenerate the uncompressed `.tar` and run `pmbootstrap checksum`.

The tarball regeneration command in the APKBUILD comment already lists
`examples`, so no change is needed there.

## Testing

### `tests/test_library.py` (new)

Unit tests for `unique_name`: no collision returns the base; one collision
returns `base-2`; consecutive collisions return the first free suffix.
Pure Python, runnable as `python3 tests/test_library.py`, matches the
existing test style (plain `assert`, `if __name__ == "__main__"`).

### `tests/test_examples.py` (new)

Locates the `examples/` directory relative to the test file, and for every
`*.txt` compiles it with `compile_script` — asserting no `DuckyScriptError`
is raised and the result is a non-empty action list. Guarantees every
shipped example actually runs. Pure Python (`duckyscript` + `keymap` only,
no Qt).

### Verification plan

1. `python3 tests/test_keymap.py && python3 tests/test_duckyscript.py &&
   python3 tests/test_library.py && python3 tests/test_examples.py` — all
   pass.
2. Offscreen GUI smoke test (`QT_QPA_PLATFORM=offscreen`) — engine loads
   `main.qml` with the new `FileDialog` and Import action, no QML errors.
3. Build with `pmbootstrap build rubberducky-pm`; `check()` runs all four
   test files.

### Untested on hardware

The `FileDialog` interaction and actual HID typing of the example payloads
require the phone attached to a USB host — consistent with the rest of the
app's status. The import logic and the examples' compilation are fully
covered by unit tests.

## Out of scope

- Editor-preview-before-save on import (user chose copy-straight-to-library).
- Runtime or build-time network downloads (user chose curate-and-commit).
- The pre-existing `README.md` build snippet using `.tar.gz` (contradicts
  the APKBUILD comment and the known crossdirect trap) — noted, not fixed.

## Files

**New:** `rubberducky_pm/library.py`, `tests/test_library.py`,
`tests/test_examples.py`, `examples/rickroll.txt`,
`examples/matrix-wake-up.txt`, `examples/windows-notepad.txt`,
`examples/linux-terminal.txt`.

**Modified:** `rubberducky_pm/app.py`, `qml/main.qml`, `APKBUILD`, and the
regenerated `rubberducky-pm-0.1.0.tar`.

**Unchanged:** `examples/hello-world.txt`.
