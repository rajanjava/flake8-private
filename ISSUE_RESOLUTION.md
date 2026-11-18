# Issue Resolution: --output-file with end of argument (--) fails with FileNotFoundError

## Metadata
- **Target Repository**: `pycqa/flake8`
- **Private Repository**: `rajanjava/flake8-private`
- **Upstream Issue**: [#1774: --output-file with end of argument (--) worked in 3.7.9 but no longer works in 6.0.0](https://github.com/PyCQA/flake8/issues/1774)
- **Status**: Resolved & Verified

---

## Problem Description & Root Cause
In `flake8.options.parse_args.parse_args()`:
```python
args0, rest = prelim_parser.parse_known_args(argv)
# XXX (ericvw): Special case "forwarding" the output file option so
# that it can be reparsed again for the BaseFormatter.filename.
if args0.output_file:
    rest.extend(("--output-file", args0.output_file))
```

When users run flake8 passing `--` to indicate the end of options (common in CI pipelines, `xargs`, or git hooks, e.g. `flake8 --output-file output.txt -- ./a`):
1. `prelim_parser.parse_known_args()` extracts `--output-file output.txt` into `args0`, leaving `rest` as `["--", "./a"]`.
2. `parse_args` then blindly appended `("--output-file", args0.output_file)` to the end of `rest`, resulting in `["--", "./a", "--output-file", "output.txt"]`.
3. Because `--` marks the end of command-line flags, `argparse` treated everything after `--` as positional arguments (filenames to lint).
4. `opts.output_file` became `None`, while `opts.filenames` included `'--output-file'` and `'output.txt'`.
5. Flake8 attempted to lint `--output-file`, producing:
   `--output-file:0:1: E902 FileNotFoundError: [Errno 2] No such file or directory: '--output-file'`

---

## Solution Approach
1. **Trace Double-Dash (`--`) Argument Parsing:** Flake8's custom `parse_args` pre-scans options (like `--output-file`) before delegating to argparse. When `--` (POSIX end-of-options marker) appeared on the command line, options appearing before `--` or filenames after `--` were misaligned, causing `FileNotFoundError`.
2. **Respect POSIX `--` Delimiter Boundaries:** Updated `src/flake8/options/parse_args.py` to stop option scanning at the `--` delimiter and preserve options before `--` separately from positional file arguments after `--`.
3. **TDD & Regression Safety:** Added `test_output_file_with_end_of_arguments` in `tests/integration/test_main.py` confirming `--output-file` works cleanly alongside `--`.

## Solution
When `"--"` is present in `rest`, insert `("--output-file", args0.output_file)` before the `"--"` marker rather than appending it to the end:
```python
if args0.output_file:
    try:
        dash_dash_idx = rest.index("--")
    except ValueError:
        rest.extend(("--output-file", args0.output_file))
    else:
        rest[dash_dash_idx:dash_dash_idx] = ("--output-file", args0.output_file)
```
This ensures `--output-file` is always parsed as an option flag rather than a positional filename argument.

---

## Code Changes
```diff
diff --git a/src/flake8/options/parse_args.py b/src/flake8/options/parse_args.py
index ff5e08f..828fdb2 100644
--- a/src/flake8/options/parse_args.py
+++ b/src/flake8/options/parse_args.py
@@ -22,7 +22,12 @@ def parse_args(
     # XXX (ericvw): Special case "forwarding" the output file option so
     # that it can be reparsed again for the BaseFormatter.filename.
     if args0.output_file:
-        rest.extend(("--output-file", args0.output_file))
+        try:
+            dash_dash_idx = rest.index("--")
+        except ValueError:
+            rest.extend(("--output-file", args0.output_file))
+        else:
+            rest[dash_dash_idx:dash_dash_idx] = ("--output-file", args0.output_file)
```

---

## Verification
- Added integration test `test_output_file_with_end_of_arguments` in `tests/integration/test_main.py`.
- Verified failure prior to fix:
  `AssertionError: assert '--output-file:0:1: E902 FileNotFoundError: ...' == ''`
- Verified pass after fix:
  `24 passed in 0.76s`
