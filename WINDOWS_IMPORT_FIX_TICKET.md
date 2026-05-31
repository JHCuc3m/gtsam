# Windows: `import gtsam` fails with circular import error

**GitHub issue:** https://github.com/borglab/gtsam/issues/2390

## Issue

On Windows with Python 3.8+, running `import gtsam` fails with:

```
ImportError: cannot import name 'gtsam' from partially initialized module 'gtsam'
(most likely due to a circular import)
(C:\...\site-packages\gtsam\__init__.py)
```

**Root cause:** This is not actually a circular import. Python 3.8 changed how
Windows finds DLL files when loading compiled Python extensions. Before 3.8,
Windows would automatically search the folder containing the extension for its
DLL dependencies. From 3.8 onwards it no longer does this.

When `import gtsam` runs, it tries to load `gtsam.pyd` (the compiled C
extension). That file depends on `gtsam.dll`, which sits right next to it in
`site-packages\gtsam\`. Windows can no longer find it there automatically, the
load fails silently, and Python reports the confusing "circular import" error
instead of a clear "DLL not found" message.

## Fix

Add one call to `os.add_dll_directory()` at the top of `gtsam\__init__.py`
to explicitly register the package folder as a DLL search location before the
extension is loaded. This is the same fix used by NumPy, OpenCV, and Pillow
for the identical issue.


---

## Steps to verify

> **Prerequisites:** Python 3.8 or later, `pip` available. No build tools or
> source checkout needed.

### Step 1 — Confirm the bug exists

Install gtsam (skip if already installed):

```powershell
pip install gtsam
```

Run the import:

```powershell
python -c "import gtsam"
```

Expected (broken) output:

```
ImportError: cannot import name 'gtsam' from partially initialized module 'gtsam' ...
```

---

### Step 2 — Apply the patch

Find where gtsam is installed:

```powershell
python -c "import site; print(site.getsitepackages()[0])"
```

This prints something like:
```
C:\Users\you\anaconda3\envs\myenv\Lib\site-packages
```

Open the file `__init__.py` in that folder with any text editor (Notepad is fine):

```
C:\Users\you\anaconda3\envs\myenv\Lib\site-packages\gtsam\__init__.py
```

Find this line near the top of the file:

```python
import sys
```

Add the following lines **directly after it**:

```python
import os

if sys.platform == "win32" and hasattr(os, "add_dll_directory"):
    os.add_dll_directory(os.path.dirname(os.path.abspath(__file__)))
```

Save the file.

---

### Step 3 — Verify the fix

```powershell
python -c "import gtsam; print('OK:', gtsam.__version__)"
```

Expected (fixed) output:

```
OK: 4.x.x
```

Run a slightly more complete smoke test to make sure the core API works:

```powershell
python -c "
import gtsam
graph = gtsam.NonlinearFactorGraph()
values = gtsam.Values()
print('NonlinearFactorGraph and Values created OK')
print('gtsam version:', gtsam.__version__)
"
```

Expected output:

```
NonlinearFactorGraph and Values created OK
gtsam version: 4.x.x
```

---

### Step 4 — Report back

Please reply with one of the following:

- **Fix confirmed** — paste the output of Step 3.
- **Still failing** — paste the full error output from Step 3 and your Python
  version (`python --version`) and gtsam version (`pip show gtsam`).
- **Bug not present** — import already worked before Step 2; include your
  Python and gtsam versions.
