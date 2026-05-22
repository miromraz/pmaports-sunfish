# Rubber Ducky — file import + bundled examples — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a file-picker import for DuckyScript `.txt` files and ship a curated set of harmless example payloads that auto-seed into the payload library on first launch.

**Architecture:** A new Qt-free `library.py` holds the pure `unique_name` dedup helper (so the abuild `check()`, which has no `py3-qt6`, can test it). `Backend` gains `importPayload`, an `examplesDir` property, and first-launch example seeding. `main.qml` gets a `FileDialog` + Import action. Example scripts live in `examples/` and are installed to `/usr/share/rubberducky-pm/examples/`.

**Tech Stack:** Python 3, PyQt6, QML/Kirigami, Alpine APKBUILD, pmbootstrap.

**Spec:** `docs/superpowers/specs/2026-05-22-rubberducky-file-import-design.md`

**Repo note:** `community/` is currently untracked in git. Commit steps `git add` only the specific files this feature touches; everything else in `community/` is left as-is. The build tarball `rubberducky-pm-0.1.0.tar` is gitignored (`*.tar`) — it is regenerated, never committed.

**Working directory for all relative paths:** `community/rubberducky-pm/` unless stated otherwise.

---

### Task 1: `unique_name` dedup helper

**Files:**
- Create: `community/rubberducky-pm/rubberducky_pm/library.py`
- Test: `community/rubberducky-pm/tests/test_library.py`

- [ ] **Step 1: Write the failing test**

Create `community/rubberducky-pm/tests/test_library.py`:

```python
"""Tests for the Qt-free payload-library helpers."""
import os
import sys

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from rubberducky_pm.library import unique_name


def test_no_collision_returns_base():
    assert unique_name("payload", []) == "payload"
    assert unique_name("payload", ["other"]) == "payload"


def test_one_collision_appends_2():
    assert unique_name("payload", ["payload"]) == "payload-2"


def test_consecutive_collisions_pick_next_free():
    existing = ["payload", "payload-2", "payload-3"]
    assert unique_name("payload", existing) == "payload-4"


def test_collision_with_gap_picks_first_free():
    assert unique_name("payload", ["payload", "payload-3"]) == "payload-2"


def test_existing_may_be_any_iterable():
    assert unique_name("payload", {"payload", "payload-2"}) == "payload-3"


if __name__ == "__main__":
    import traceback

    tests = sorted(
        (k, v) for k, v in globals().items() if k.startswith("test_") and callable(v)
    )
    failed = 0
    for name, fn in tests:
        try:
            fn()
            print(f"PASS {name}")
        except Exception as exc:  # noqa: BLE001 - test harness
            failed += 1
            print(f"FAIL {name}: {exc}")
            traceback.print_exc()
    print(f"\n{len(tests) - failed}/{len(tests)} passed")
    sys.exit(1 if failed else 0)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd community/rubberducky-pm && python3 tests/test_library.py`
Expected: FAIL — `ModuleNotFoundError: No module named 'rubberducky_pm.library'`

- [ ] **Step 3: Write minimal implementation**

Create `community/rubberducky-pm/rubberducky_pm/library.py`:

```python
"""Qt-free helpers for the payload library.

Kept free of any Qt import so the abuild ``check()`` environment -- which
has ``python3`` but not ``py3-qt6`` -- can unit-test it.
"""


def unique_name(base, existing):
    """Return ``base``, or the first free ``base-2`` / ``base-3`` / ... .

    ``existing`` is any iterable of payload names already in the library.
    Used so importing a file never silently overwrites an existing payload.
    """
    taken = set(existing)
    if base not in taken:
        return base
    suffix = 2
    while "%s-%d" % (base, suffix) in taken:
        suffix += 1
    return "%s-%d" % (base, suffix)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd community/rubberducky-pm && python3 tests/test_library.py`
Expected: PASS — `5/5 passed`

- [ ] **Step 5: Commit**

```bash
cd /home/realni/pmaports-sm7150
git add community/rubberducky-pm/rubberducky_pm/library.py community/rubberducky-pm/tests/test_library.py
git commit -m "feat(rubberducky): add unique_name payload-dedup helper"
```

---

### Task 2: Bundled example scripts

**Files:**
- Create: `community/rubberducky-pm/examples/rickroll.txt`
- Create: `community/rubberducky-pm/examples/windows-notepad.txt`
- Create: `community/rubberducky-pm/examples/linux-terminal.txt`
- Create: `community/rubberducky-pm/examples/matrix-wake-up.txt`
- Test: `community/rubberducky-pm/tests/test_examples.py`
- Unchanged: `community/rubberducky-pm/examples/hello-world.txt`

- [ ] **Step 1: Write the failing test**

Create `community/rubberducky-pm/tests/test_examples.py`:

```python
"""Every bundled example payload must compile under the supported subset."""
import os
import sys

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from rubberducky_pm.duckyscript import compile_script

EXAMPLES_DIR = os.path.join(
    os.path.dirname(os.path.dirname(os.path.abspath(__file__))), "examples"
)


def _example_files():
    names = sorted(
        n for n in os.listdir(EXAMPLES_DIR) if n.endswith(".txt")
    )
    assert names, "no example .txt files found in %s" % EXAMPLES_DIR
    return names


def test_every_example_compiles_to_actions():
    for name in _example_files():
        path = os.path.join(EXAMPLES_DIR, name)
        with open(path, "r", encoding="utf-8") as handle:
            text = handle.read()
        actions = compile_script(text)
        assert actions, "%s compiled to no actions" % name


if __name__ == "__main__":
    import traceback

    tests = sorted(
        (k, v) for k, v in globals().items() if k.startswith("test_") and callable(v)
    )
    failed = 0
    for name, fn in tests:
        try:
            fn()
            print(f"PASS {name}")
        except Exception as exc:  # noqa: BLE001 - test harness
            failed += 1
            print(f"FAIL {name}: {exc}")
            traceback.print_exc()
    print(f"\n{len(tests) - failed}/{len(tests)} passed")
    sys.exit(1 if failed else 0)
```

- [ ] **Step 2: Run test to verify it passes for the existing example**

Run: `cd community/rubberducky-pm && python3 tests/test_examples.py`
Expected: PASS — `1/1 passed` (only `hello-world.txt` exists so far; this confirms the harness works before new files are added).

- [ ] **Step 3: Create `examples/rickroll.txt`**

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

- [ ] **Step 4: Create `examples/windows-notepad.txt`**

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

- [ ] **Step 5: Create `examples/linux-terminal.txt`**

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

- [ ] **Step 6: Create `examples/matrix-wake-up.txt`**

Adapted from the Hak5 payload `payloads/library/prank/The_Matrix-Wake_Up/payload.txt` by UberGuidoZ. The `ATTACKMODE HID STORAGE` line (DuckyScript 3.0, unsupported) is removed; one `REM` line records the adaptation. Content verbatim:

```
REM Title: The Matrix Wake Up
REM Description: Recreates the Wake Up Neo terminal scene in The Matrix
REM Author: UberGuidoZ
REM Adapted for rubberducky-pm: removed ATTACKMODE (DuckyScript 3.0, unsupported)
REM Target: Windows (including Powershell 2.0 or above)
REM Version: v1.1
DELAY 3000
GUI r
DELAY 750
STRING cmd
ENTER
DELAY 750
STRING color 02 && ECHO OFF && cls
ENTER
ALT ENTER
DELAY 1000
STRING W
DELAY 100
STRING a
DELAY 100
STRING k
DELAY 100
STRING e
DELAY 100
SPACE
DELAY 100
STRING u
DELAY 100
STRING p
DELAY 100
STRING .
DELAY 100
SPACE
DELAY 1000
STRING N
DELAY 250
STRING e
DELAY 250
STRING o
DELAY 250
STRING .
DELAY 250
STRING .
DELAY 250
STRING .
DELAY 3500
CTRL HOME
DELAY 1500
STRING T
DELAY 300
STRING h
DELAY 300
STRING e
DELAY 300
SPACE
DELAY 300
STRING M
DELAY 300
STRING a
DELAY 300
STRING t
DELAY 300
STRING r
DELAY 300
STRING i
DELAY 300
STRING x
DELAY 300
SPACE
DELAY 300
STRING h
DELAY 300
STRING a
DELAY 300
STRING s
DELAY 300
SPACE
DELAY 300
STRING y
DELAY 300
STRING o
DELAY 300
STRING u
DELAY 300
STRING .
DELAY 300
STRING .
DELAY 300
STRING .
DELAY 3500
CTRL HOME
STRING F
DELAY 100
STRING o
DELAY 100
STRING l
DELAY 100
STRING l
DELAY 100
STRING o
DELAY 100
STRING w
DELAY 100
SPACE
DELAY 100
STRING t
DELAY 100
STRING h
DELAY 100
STRING e
DELAY 100
SPACE
DELAY 100
STRING w
DELAY 100
STRING h
DELAY 100
STRING i
DELAY 100
STRING t
DELAY 100
STRING e
DELAY 100
SPACE
DELAY 100
STRING r
DELAY 100
STRING a
DELAY 100
STRING b
DELAY 100
STRING b
DELAY 100
STRING i
DELAY 100
STRING t
DELAY 100
STRING .
DELAY 3500
CTRL HOME
DELAY 1500
STRING Knock, knock, Neo.
DELAY 3500
CTRL HOME
STRING COLOR 7F
ENTER
ALT ENTER
STRING mode con:cols=18 lines=1
ENTER
STRING powershell [console]::beep(200,325); [console]::beep(200,325)
ENTER
DELAY 1500
ALT F4
```

- [ ] **Step 7: Run the test to verify all five examples compile**

Run: `cd community/rubberducky-pm && python3 tests/test_examples.py`
Expected: PASS — `1/1 passed` (one test function, now iterating all 5 files). If it FAILS, the error names the offending file and line — fix that example so it stays within the supported DuckyScript 1.0 subset, then re-run.

- [ ] **Step 8: Commit**

```bash
cd /home/realni/pmaports-sm7150
git add community/rubberducky-pm/examples community/rubberducky-pm/tests/test_examples.py
git commit -m "feat(rubberducky): add curated example payloads + compile test"
```

---

### Task 3: Backend — import slot, examples dir, first-launch seeding

**Files:**
- Modify: `community/rubberducky-pm/rubberducky_pm/app.py`

- [ ] **Step 1: Add imports**

In `app.py`, change the stdlib import block (currently `import os`, `import sys`, `import tempfile`) to add `shutil`:

```python
import os
import shutil
import sys
import tempfile
```

After the `from .duckyscript import ...` line, add:

```python
from .library import unique_name
```

- [ ] **Step 2: Add the `_examples_dir()` module function**

Immediately after the existing `_resource_dir()` function (before `def main`), add:

```python
def _examples_dir():
    """Locate the directory holding the bundled example payloads."""
    override = os.environ.get("RUBBERDUCKY_EXAMPLES_DIR")
    if override and os.path.isdir(override):
        return override
    installed = "/usr/share/rubberducky-pm/examples"
    if os.path.isdir(installed):
        return installed
    local = os.path.join(
        os.path.dirname(os.path.abspath(__file__)), os.pardir, "examples"
    )
    if os.path.isdir(local):
        return local
    return None
```

- [ ] **Step 3: Seed examples on first launch**

In `Backend.__init__`, the current tail is:

```python
        self._helper = os.environ.get("RUBBERDUCKY_HELPER", HELPER_DEFAULT)
        self._rescan()
```

Replace it with:

```python
        self._helper = os.environ.get("RUBBERDUCKY_HELPER", HELPER_DEFAULT)
        if not self._settings.value("examplesSeeded", False, type=bool):
            self._seed_examples()
            self._settings.setValue("examplesSeeded", True)
        self._rescan()
```

- [ ] **Step 4: Add the `_seed_examples` method**

In the `# --- payload storage ---` section, after `_path_for`, add:

```python
    def _seed_examples(self):
        """Copy bundled examples into the library (first launch only).

        Best-effort: a name already in the library is left untouched, and any
        I/O failure is logged but never aborts startup.
        """
        src_dir = _examples_dir()
        if not src_dir:
            return
        try:
            files = os.listdir(src_dir)
        except OSError:
            return
        for fname in files:
            if not fname.endswith(".txt"):
                continue
            dest = self._path_for(os.path.splitext(fname)[0])
            if os.path.exists(dest):
                continue
            try:
                shutil.copyfile(os.path.join(src_dir, fname), dest)
            except OSError as exc:
                sys.stderr.write(
                    "rubberducky: could not seed %s: %s\n" % (fname, exc)
                )
```

- [ ] **Step 5: Add the `examplesDir` property**

In the `# --- properties ---` section, after the `payloadNames` property, add:

```python
    @Property(QUrl, constant=True)
    def examplesDir(self):
        """Folder the import file-picker should open in."""
        path = _examples_dir()
        if path is None:
            path = QStandardPaths.writableLocation(
                QStandardPaths.StandardLocation.DownloadLocation
            ) or os.path.expanduser("~")
        return QUrl.fromLocalFile(path)
```

- [ ] **Step 6: Add the `importPayload` slot**

In the `# --- payload storage ---` section, after `deletePayload`, add:

```python
    @Slot(QUrl, result=str)
    def importPayload(self, url):
        """Copy a DuckyScript file into the library; return its new name.

        Returns "" if the file is unreadable, not UTF-8 text, or empty.
        The name is derived from the filename and de-duplicated so an
        import never overwrites an existing payload.
        """
        path = url.toLocalFile()
        if not path:
            return ""
        try:
            with open(path, "r", encoding="utf-8") as handle:
                text = handle.read()
        except (OSError, UnicodeDecodeError):
            return ""
        if not text.strip():
            return ""
        base = os.path.splitext(os.path.basename(path))[0].strip()
        if not base:
            base = "imported"
        name = unique_name(base, self._names)
        try:
            with open(self._path_for(name), "w", encoding="utf-8") as handle:
                handle.write(text)
        except OSError:
            return ""
        self._rescan()
        return name
```

- [ ] **Step 7: Verify the file is syntactically valid**

Run: `cd community/rubberducky-pm && python3 -m py_compile rubberducky_pm/app.py`
Expected: no output, exit 0 (compiles clean — `py_compile` does not import PyQt6, so it works without Qt installed).

- [ ] **Step 8: Commit**

```bash
cd /home/realni/pmaports-sm7150
git add community/rubberducky-pm/rubberducky_pm/app.py
git commit -m "feat(rubberducky): backend file import + first-launch example seeding"
```

---

### Task 4: QML — Import action and file dialog

**Files:**
- Modify: `community/rubberducky-pm/qml/main.qml`

- [ ] **Step 1: Add the `QtQuick.Dialogs` import**

In `main.qml`, the import block is:

```qml
import QtQuick
import QtQuick.Controls as QQC2
import QtQuick.Layouts
import org.kde.kirigami as Kirigami
```

Add one line:

```qml
import QtQuick
import QtQuick.Controls as QQC2
import QtQuick.Layouts
import QtQuick.Dialogs
import org.kde.kirigami as Kirigami
```

- [ ] **Step 2: Add the Import action to the list page**

The `listPage` `actions` array currently holds only the "New" action. Replace the whole `actions` block with:

```qml
        actions: [
            Kirigami.Action {
                text: "New"
                icon.name: "list-add"
                enabled: !Backend.running
                onTriggered: root.openEditor("", "")
            },
            Kirigami.Action {
                text: "Import"
                icon.name: "document-import"
                enabled: !Backend.running
                onTriggered: importDialog.open()
            }
        ]
```

- [ ] **Step 3: Add the `FileDialog`**

After the `aboutSheet` `Kirigami.OverlaySheet { ... }` block and before the `Connections { target: Backend ... }` block, add:

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

- [ ] **Step 4: Smoke-test the GUI offscreen**

Run (from `community/rubberducky-pm/`):

```bash
timeout 8 env QT_QPA_PLATFORM=offscreen RUBBERDUCKY_QML_DIR=./qml \
  RUBBERDUCKY_HELPER=$PWD/bin/rubberducky-pm-helper \
  python3 -m rubberducky_pm 2>&1 | tee /tmp/ducky-smoke.log; \
  grep -iE 'error|cannot|undefined' /tmp/ducky-smoke.log || echo "NO QML ERRORS"
```

Expected: prints `NO QML ERRORS`. The window loads headless and `timeout` ends it after 8 s.
Contingency: if this fails with `ModuleNotFoundError: PyQt6` or an unresolved `org.kde.kirigami` import, the local machine lacks `python-pyqt6`/Kirigami — note it and rely on the on-device verification in Task 6. A QML *syntax* error (mismatched braces, bad property) will still surface in the log even when modules are missing, so review the log either way.

- [ ] **Step 5: Commit**

```bash
cd /home/realni/pmaports-sm7150
git add community/rubberducky-pm/qml/main.qml
git commit -m "feat(rubberducky): Import action + file picker in the UI"
```

---

### Task 5: APKBUILD — install examples, run new tests, rebuild

**Files:**
- Modify: `community/rubberducky-pm/APKBUILD`
- Regenerate (not committed — gitignored): `community/rubberducky-pm/rubberducky-pm-0.1.0.tar`

- [ ] **Step 1: Bump `pkgrel`**

In `APKBUILD`, change `pkgrel=3` to `pkgrel=4`.

- [ ] **Step 2: Add the new tests to `check()`**

Replace the `check()` function with:

```sh
check() {
	python3 "$builddir"/tests/test_keymap.py
	python3 "$builddir"/tests/test_duckyscript.py
	python3 "$builddir"/tests/test_library.py
	python3 "$builddir"/tests/test_examples.py
}
```

- [ ] **Step 3: Install the examples in `package()`**

In `package()`, the `mkdir -p` call currently is:

```sh
	mkdir -p "$share/rubberducky_pm" "$share/qml" \
		"$pkgdir/usr/bin" \
		"$pkgdir/usr/share/applications" \
		"$pkgdir/usr/share/polkit-1/actions"
```

Add `"$share/examples"`:

```sh
	mkdir -p "$share/rubberducky_pm" "$share/qml" "$share/examples" \
		"$pkgdir/usr/bin" \
		"$pkgdir/usr/share/applications" \
		"$pkgdir/usr/share/polkit-1/actions"
```

Then, immediately after the `install -m644 "$builddir"/qml/*.qml "$share/qml/"` line, add:

```sh
	install -m644 "$builddir"/examples/*.txt "$share/examples/"
```

- [ ] **Step 4: Regenerate the source tarball**

Run from `community/rubberducky-pm/`:

```bash
cd /home/realni/pmaports-sm7150/community/rubberducky-pm
rm -f rubberducky-pm-0.1.0.tar
tar cf rubberducky-pm-0.1.0.tar \
    --transform "s,^,rubberducky-pm-0.1.0/," \
    rubberducky_pm qml bin data tests examples README.md
```

Expected: `rubberducky-pm-0.1.0.tar` created, no errors. (`.tar`, never `.tar.gz` — abuild's gzip unpack path is broken under crossdirect.)

- [ ] **Step 5: Update the checksum**

Run: `pmbootstrap checksum rubberducky-pm`
Expected: the `sha512sums=` block in `APKBUILD` is rewritten with the new tar hash.

- [ ] **Step 6: Build the package (runs `check()`)**

Run: `pmbootstrap build rubberducky-pm`
Expected: build succeeds; the log shows all four test files run and report `passed` with exit 0. If `check()` fails, fix the offending test/code and rebuild before continuing.

- [ ] **Step 7: Commit**

```bash
cd /home/realni/pmaports-sm7150
git add community/rubberducky-pm/APKBUILD
git commit -m "rubberducky-pm: install examples, run new tests, pkgrel 4"
```

(The regenerated `.tar` is gitignored and intentionally not committed.)

---

### Task 6: Deploy to the phone and verify on device

**Not a TDD task — deployment + behavioural verification.** Requires the Pixel 4a reachable over SSH (see the `sunfish-phone-access` memory for host/credentials).

- [ ] **Step 1: Locate the built apk**

Run: `find ~/.local/var/pmbootstrap/packages -name 'rubberducky-pm-0.1.0-r4*.apk'`
Expected: one `noarch` apk path.

- [ ] **Step 2: Copy and install on the phone**

`scp` the apk to the phone, then on the phone run `apk add --allow-untrusted /path/to/rubberducky-pm-0.1.0-r4.apk`.
Expected: upgrades `rubberducky-pm-0.1.0-r3` → `r4`.

- [ ] **Step 3: Launch and verify the examples seeded**

Launch Rubber Ducky on the phone (the launcher already exists from r3 — no `kbuildsycoca` needed for an upgrade). 
Expected: the Payloads list shows the five examples — `hello-world`, `linux-terminal`, `matrix-wake-up`, `rickroll`, `windows-notepad` — on first launch after the upgrade.

- [ ] **Step 4: Verify the import action**

Tap **Import** in the toolbar. Expected: a file picker opens at `/usr/share/rubberducky-pm/examples`. Pick any `.txt`; expect a passive notification `Imported as <name>` and the file to appear in the list (with a `-2` suffix if the name already existed).

- [ ] **Step 5: Manual end-to-end run (user-driven)**

Plug the phone into a test computer and tap **Run** on `linux-terminal` (Linux host) or `windows-notepad` (Windows host). This step needs a physical USB host and cannot be observed over SSH (dedicated mode drops USB-net). This is the real "running real scripts" check and is performed manually by the user.

---

## Self-review

**Spec coverage:** file-picker import → Tasks 3 (`importPayload`) + 4 (`FileDialog`/action); auto-seed → Task 3 (`_seed_examples`, `examplesSeeded` flag); `examplesDir` → Task 3; five example scripts → Task 2; `unique_name` in Qt-free `library.py` → Task 1; APKBUILD install + `check()` + `pkgrel` → Task 5; `test_library.py` + `test_examples.py` → Tasks 1, 2; verification plan → Tasks 4–6. All spec sections mapped.

**Placeholder scan:** no TBD/TODO; every code step shows complete content including the full `matrix-wake-up.txt`.

**Type consistency:** `unique_name(base, existing)` defined in Task 1, called identically in Task 3. `importPayload` / `examplesDir` / `_examples_dir` / `_seed_examples` names consistent across Tasks 3 and 4 and the QML (`Backend.importPayload`, `Backend.examplesDir`). `examplesSeeded` settings key used once. Test filenames `test_library.py` / `test_examples.py` consistent between Tasks 1/2 and the `check()` block in Task 5.
