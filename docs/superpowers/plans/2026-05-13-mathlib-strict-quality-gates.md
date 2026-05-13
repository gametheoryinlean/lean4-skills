# Mathlib-Strict Quality Gates Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a minimal deterministic `mathlib-strict` quality gate for Lean code rewrites so `lean4-skills` can reject AI-generated diffs that fall below Mathlib-style safety and review standards.

**Architecture:** Implement one host-agnostic Python gate runner under `plugins/lean4/lib/scripts/` and wire it into existing review/refactor/golf documentation. The runner compares current Lean files against a baseline, emits JSON and Markdown reports, rejects hard safety/API drift, and flags controlled changes for human review.

**Tech Stack:** Python standard library, `unittest`, existing Lean script layout, existing Markdown command docs, optional git baseline via `git show HEAD:<path>`.

---

## Official Mathlib Basis

This plan encodes only requirements that are backed by Mathlib/Lean community documentation:

- Mathlib contribution baseline: code must build, avoid `sorry`, pass CI/review, and AI-generated contributions remain the author's responsibility. Source: <https://leanprover-community.github.io/contribute/index.html>
- Style requirements: 100-character line width, file/module structure, import grouping, explicit declarations, indentation, and API/performance care. Source: <https://leanprover-community.github.io/contribute/style.html>
- Naming conventions: theorem names use snake case, type/class/structure names use upper camel case, ordinary term names use lower camel case, and names should follow Mathlib conventions. Source: <https://leanprover-community.github.io/contribute/naming.html>
- Documentation expectations: module docstrings, definition/theorem docstrings where appropriate, and doc linting tools. Source: <https://leanprover-community.github.io/contribute/doc.html>
- PR review dimensions: existing lemmas, API fit, imports, file location, proof decomposition, documentation, instances, and performance risk. Source: <https://leanprover-community.github.io/contribute/pr-review.html>
- Mathlib commands: `lake build`, `lake test`, module builds, and `lake exe mk_all` for new files. Source: <https://github.com/leanprover-community/mathlib4>

## File Structure

Create and modify these files:

- Create: `plugins/lean4/lib/scripts/mathlib_strict_gate.py`
  - Responsibility: static diff-aware gate runner for the `mathlib-strict` profile.
- Create: `plugins/lean4/lib/scripts/tests/test_mathlib_strict_gate.py`
  - Responsibility: unit tests for hard rejects, human-review classifications, and JSON schema.
- Modify: `plugins/lean4/lib/scripts/README.md`
  - Responsibility: document the new script interface and exit behavior.
- Modify: `plugins/lean4/lib/scripts/TESTING.md`
  - Responsibility: add the new script and its test command to script validation docs.
- Modify: `plugins/lean4/commands/review.md`
  - Responsibility: describe the strict gate report in batch review mode.
- Modify: `plugins/lean4/commands/refactor.md`
  - Responsibility: make mutating refactors run the strict static gate after Lean verification when targeting Mathlib-style cleanup.
- Modify: `plugins/lean4/commands/golf.md`
  - Responsibility: make proof-golfing safety mention header/API drift guards.
- Modify: `plugins/lean4/skills/lean4/SKILL.md`
  - Responsibility: mention the script in the quality gate/quality references section.
- Modify: `plugins/lean4/skills/lean4/references/mathlib-quality-system.md`
  - Responsibility: update the design note from "proposed only" to "v1 static gate runner exists", while preserving the warning that full Mathlib quality still needs build/lint/human review.

Do not add a new slash command in v1. The gate runner is a reusable primitive that existing review/refactor/golf flows and external runners can call.

## Gate Contract

`mathlib_strict_gate.py` supports exactly one profile in v1:

```text
mathlib-strict
```

CLI:

```bash
${LEAN4_PYTHON_BIN:-python3} "$LEAN4_SCRIPTS/mathlib_strict_gate.py" [TARGET ...] \
  --baseline-dir PATH \
  --report-json mathlib-strict-report.json \
  --report-md mathlib-strict-report.md
```

Default behavior when `TARGET` is omitted:

- Use changed `.lean` files from `git diff --name-only HEAD -- '*.lean'` and `git diff --cached --name-only -- '*.lean'`.
- If no git baseline is available, require explicit targets and `--baseline-dir`.

Exit behavior:

- Exit `1` when hard status is `reject`.
- Exit `0` for `accept` and `needs-human-review`.
- `--report-only` forces exit `0` for findings but still exits nonzero for real errors.
- `--fail-on-review` also exits `1` for `needs-human-review`.

Status meanings:

- `accept`: no hard failures and no controlled-review findings.
- `needs-human-review`: imports, missing docs, naming/style warnings, or other controlled items need review.
- `reject`: hard safety/API gates failed.

Hard rejects:

- New `sorry` or `admit` tokens in code.
- New source-level `axiom` or `unsafe` tokens in code.
- Public declaration header changed, added, or removed without `--allow-public-api-changes`.

Human-review findings:

- Added imports.
- Lines over 100 characters.
- Missing module docstring in a touched file.
- Public declarations without nearby docstrings.
- Obvious naming convention mismatch.

This v1 script is static and diff-aware. It complements, but does not replace:

```bash
lake env lean path/to/File.lean
lake build
lake test
lake exe lint-style
lake exe lintAll
```

### Task 1: Add Failing Tests For The Gate Runner

**Files:**
- Create: `plugins/lean4/lib/scripts/tests/test_mathlib_strict_gate.py`
- Expected red result before implementation: test process fails because `plugins/lean4/lib/scripts/mathlib_strict_gate.py` does not exist.

- [ ] **Step 1: Write the failing test file**

Use `apply_patch` to add this file:

```python
#!/usr/bin/env python3
"""Tests for mathlib_strict_gate.py.

Run from repo root:
    python3 plugins/lean4/lib/scripts/tests/test_mathlib_strict_gate.py
"""

import json
import subprocess
import sys
import tempfile
import unittest
from pathlib import Path


SCRIPT = Path(__file__).resolve().parents[1] / "mathlib_strict_gate.py"


BASELINE = """\
import Mathlib

/-!
# Demo

A tiny module docstring.
-/

/-- Adding zero on the right. -/
theorem add_zero_demo (n : Nat) : n + 0 = n := by
  simpa
"""


PROOF_BODY_CHANGED = """\
import Mathlib

/-!
# Demo

A tiny module docstring.
-/

/-- Adding zero on the right. -/
theorem add_zero_demo (n : Nat) : n + 0 = n := by
  exact Nat.add_zero n
"""


STATEMENT_CHANGED = """\
import Mathlib

/-!
# Demo

A tiny module docstring.
-/

/-- Adding zero on the left. -/
theorem add_zero_demo (n : Nat) : 0 + n = n := by
  simpa
"""


WITH_REAL_SORRY = """\
import Mathlib

/-!
# Demo

A tiny module docstring.
-/

/-- Adding zero on the right. -/
theorem add_zero_demo (n : Nat) : n + 0 = n := by
  sorry
"""


WITH_COMMENT_AND_STRING_SORRY = """\
import Mathlib

/-!
# Demo

A tiny module docstring.
-/

def sorryText : String := "sorry"

-- sorry in comments is not a proof hole.
/-- Adding zero on the right. -/
theorem add_zero_demo (n : Nat) : n + 0 = n := by
  simpa
"""


WITH_ADDED_IMPORT = """\
import Mathlib
import Mathlib.Algebra.Group.Basic

/-!
# Demo

A tiny module docstring.
-/

/-- Adding zero on the right. -/
theorem add_zero_demo (n : Nat) : n + 0 = n := by
  simpa
"""


WITH_NEW_PUBLIC_DEF = """\
import Mathlib

/-!
# Demo

A tiny module docstring.
-/

/-- Adding zero on the right. -/
theorem add_zero_demo (n : Nat) : n + 0 = n := by
  simpa

/-- A new public API surface. -/
def publicValue : Nat := 1
"""


WITH_NEW_PRIVATE_DEF = """\
import Mathlib

/-!
# Demo

A tiny module docstring.
-/

private def localValue : Nat := 1

/-- Adding zero on the right. -/
theorem add_zero_demo (n : Nat) : n + 0 = n := by
  simpa
"""


class MathlibStrictGateTest(unittest.TestCase):
    def run_gate(self, current_text, baseline_text=BASELINE, extra_args=None):
        extra_args = extra_args or []
        with tempfile.TemporaryDirectory() as tmp:
            tmpdir = Path(tmp)
            repo = tmpdir / "repo"
            baseline = tmpdir / "baseline"
            repo.mkdir()
            baseline.mkdir()
            (repo / "Demo.lean").write_text(current_text, encoding="utf-8")
            (baseline / "Demo.lean").write_text(baseline_text, encoding="utf-8")
            report = tmpdir / "report.json"

            cmd = [
                sys.executable,
                str(SCRIPT),
                "Demo.lean",
                "--baseline-dir",
                str(baseline),
                "--report-json",
                str(report),
                *extra_args,
            ]
            proc = subprocess.run(cmd, cwd=repo, text=True, capture_output=True)
            data = json.loads(report.read_text(encoding="utf-8")) if report.exists() else None
            return proc, data

    def test_proof_body_rewrite_is_accepted(self):
        proc, data = self.run_gate(PROOF_BODY_CHANGED)
        self.assertEqual(proc.returncode, 0, proc.stderr + proc.stdout)
        self.assertEqual(data["status"], "accept")

    def test_public_statement_change_is_rejected(self):
        proc, data = self.run_gate(STATEMENT_CHANGED)
        self.assertEqual(proc.returncode, 1)
        self.assertEqual(data["status"], "reject")
        self.assertTrue(any(item["gate"] == "public_api_fingerprint_guard" for item in data["issues"]))

    def test_new_sorry_is_rejected(self):
        proc, data = self.run_gate(WITH_REAL_SORRY)
        self.assertEqual(proc.returncode, 1)
        self.assertEqual(data["status"], "reject")
        self.assertTrue(any(item["gate"] == "no_new_sorry_admit" for item in data["issues"]))

    def test_sorry_in_comment_and_string_is_ignored(self):
        proc, data = self.run_gate(WITH_COMMENT_AND_STRING_SORRY)
        self.assertEqual(proc.returncode, 0, proc.stderr + proc.stdout)
        self.assertNotEqual(data["status"], "reject")
        self.assertFalse(any(item["gate"] == "no_new_sorry_admit" for item in data["issues"]))

    def test_added_import_requires_human_review_but_does_not_reject(self):
        proc, data = self.run_gate(WITH_ADDED_IMPORT)
        self.assertEqual(proc.returncode, 0, proc.stderr + proc.stdout)
        self.assertEqual(data["status"], "needs-human-review")
        self.assertTrue(any(item["gate"] == "import_guard" for item in data["issues"]))

    def test_new_public_declaration_is_rejected(self):
        proc, data = self.run_gate(WITH_NEW_PUBLIC_DEF)
        self.assertEqual(proc.returncode, 1)
        self.assertEqual(data["status"], "reject")
        self.assertTrue(any(item["gate"] == "public_api_fingerprint_guard" for item in data["issues"]))

    def test_new_private_helper_is_accepted(self):
        proc, data = self.run_gate(WITH_NEW_PRIVATE_DEF)
        self.assertEqual(proc.returncode, 0, proc.stderr + proc.stdout)
        self.assertEqual(data["status"], "accept")

    def test_report_only_keeps_zero_exit_on_reject(self):
        proc, data = self.run_gate(WITH_REAL_SORRY, extra_args=["--report-only"])
        self.assertEqual(proc.returncode, 0, proc.stderr + proc.stdout)
        self.assertEqual(data["status"], "reject")


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run the test and verify RED**

Run:

```bash
python3 plugins/lean4/lib/scripts/tests/test_mathlib_strict_gate.py
```

Expected: FAIL because `mathlib_strict_gate.py` is missing or not executable by Python.

### Task 2: Implement `mathlib_strict_gate.py`

**Files:**
- Create: `plugins/lean4/lib/scripts/mathlib_strict_gate.py`
- Test: `plugins/lean4/lib/scripts/tests/test_mathlib_strict_gate.py`

- [ ] **Step 1: Add the script**

Use `apply_patch` to add this file. Keep it Python-stdlib only.

```python
#!/usr/bin/env python3
"""Static mathlib-strict gate for Lean file rewrites.

This script is intentionally conservative. It catches deterministic reject
conditions and review-worthy changes before slower Lean/lint gates run.
"""

import argparse
import json
import re
import subprocess
import sys
from pathlib import Path


VERSION = "1.0"
PROFILE = "mathlib-strict"
STATUS_ORDER = {"accept": 0, "needs-human-review": 1, "reject": 2}
TOKEN_RE = re.compile(r"(?<![A-Za-z0-9_!?'])({})(?![A-Za-z0-9_!?'])")
IMPORT_RE = re.compile(r"^\s*(public\s+)?import\s+(.+?)\s*$")
DECL_RE = re.compile(
    r"^\s*(?P<mods>(?:(?:private|protected|noncomputable|unsafe|partial)\s+)*)"
    r"(?P<kind>theorem|lemma|def|abbrev|instance|structure|class|inductive)\s+"
    r"(?P<name>[A-Za-z_][A-Za-z0-9_'.!?]*)?"
)
DOC_RE = re.compile(r"^\s*/--")


def strip_line_comments_strings(line, block_depth):
    result = []
    i = 0
    in_string = False
    while i < len(line):
        ch = line[i]
        nxt = line[i + 1] if i + 1 < len(line) else ""

        if block_depth > 0:
            if ch == "/" and nxt == "-":
                block_depth += 1
                i += 2
                continue
            if ch == "-" and nxt == "/":
                block_depth -= 1
                i += 2
                continue
            i += 1
            continue

        if in_string:
            if ch == "\\" and i + 1 < len(line):
                i += 2
                continue
            if ch == '"':
                in_string = False
            i += 1
            continue

        if ch == '"':
            in_string = True
            i += 1
            continue
        if ch == "-" and nxt == "-":
            break
        if ch == "/" and nxt == "-":
            block_depth += 1
            i += 2
            continue

        result.append(ch)
        i += 1
    return "".join(result), block_depth


def code_only_lines(text):
    depth = 0
    out = []
    for line in text.splitlines():
        stripped, depth = strip_line_comments_strings(line, depth)
        out.append(stripped)
    return out


def token_counts(text, tokens):
    pattern = TOKEN_RE.pattern.format("|".join(re.escape(t) for t in tokens))
    regex = re.compile(pattern)
    counts = {token: 0 for token in tokens}
    for line in code_only_lines(text):
        for match in regex.finditer(line):
            counts[match.group(1)] += 1
    return counts


def parse_imports(text):
    imports = set()
    for line in code_only_lines(text):
        match = IMPORT_RE.match(line)
        if match:
            prefix = "public import" if match.group(1) else "import"
            imports.add(f"{prefix} {match.group(2).strip()}")
    return imports


def normalize_header(header):
    return " ".join(header.split())


def header_until_body(lines, start):
    pieces = []
    for idx in range(start, min(len(lines), start + 40)):
        part = lines[idx].strip()
        if not part:
            continue
        pieces.append(part)
        joined = " ".join(pieces)
        for marker in (":= by", ":=", " where"):
            if marker in joined:
                return normalize_header(joined.split(marker)[0])
    return normalize_header(" ".join(pieces))


def has_nearby_docstring(raw_lines, idx):
    for pos in range(max(0, idx - 5), idx):
        if DOC_RE.match(raw_lines[pos]):
            return True
    return False


def public_declaration_fingerprints(text):
    raw_lines = text.splitlines()
    code_lines = code_only_lines(text)
    fingerprints = {}
    docs = {}
    for idx, line in enumerate(code_lines):
        match = DECL_RE.match(line)
        if not match:
            continue
        mods = (match.group("mods") or "").split()
        kind = match.group("kind")
        name = match.group("name") or f"anonymous_{kind}_{idx + 1}"
        if "private" in mods:
            continue
        header = header_until_body(code_lines, idx)
        key = f"{kind} {name}"
        fingerprints[key] = {
            "kind": kind,
            "name": name,
            "line": idx + 1,
            "header": header,
        }
        docs[key] = has_nearby_docstring(raw_lines, idx)
    return fingerprints, docs


def is_snake_case(name):
    return bool(re.match(r"^[a-z][a-z0-9_']*$", name))


def is_upper_camel(name):
    return bool(re.match(r"^[A-Z][A-Za-z0-9_']*$", name))


def status_join(current, new_status):
    return new_status if STATUS_ORDER[new_status] > STATUS_ORDER[current] else current


def issue(path, gate, status, message, line=None, details=None):
    item = {
        "file": str(path),
        "gate": gate,
        "status": status,
        "message": message,
    }
    if line is not None:
        item["line"] = line
    if details:
        item["details"] = details
    return item


def read_git_baseline(path):
    try:
        result = subprocess.run(
            ["git", "show", f"HEAD:{path.as_posix()}"],
            text=True,
            capture_output=True,
            check=False,
        )
    except OSError:
        return None
    if result.returncode != 0:
        return None
    return result.stdout


def read_baseline(path, baseline_dir):
    if baseline_dir is not None:
        baseline_path = baseline_dir / path
        if baseline_path.exists():
            return baseline_path.read_text(encoding="utf-8")
        return None
    return read_git_baseline(path)


def expand_targets(targets):
    files = []
    for raw in targets:
        path = Path(raw)
        if path.is_dir():
            files.extend(sorted(p for p in path.rglob("*.lean") if ".lake" not in p.parts))
        elif path.is_file() and path.suffix == ".lean":
            files.append(path)
    return sorted(dict.fromkeys(files))


def changed_lean_files_from_git():
    files = set()
    for args in (
        ["git", "diff", "--name-only", "HEAD", "--", "*.lean"],
        ["git", "diff", "--cached", "--name-only", "--", "*.lean"],
    ):
        result = subprocess.run(args, text=True, capture_output=True, check=False)
        if result.returncode == 0:
            files.update(line.strip() for line in result.stdout.splitlines() if line.strip())
    return [Path(f) for f in sorted(files) if Path(f).exists()]


def analyze_file(path, baseline_text, allow_public_api_changes):
    current_text = path.read_text(encoding="utf-8")
    issues = []

    baseline_sorry = token_counts(baseline_text or "", ["sorry", "admit"])
    current_sorry = token_counts(current_text, ["sorry", "admit"])
    for token in ("sorry", "admit"):
        delta = current_sorry[token] - baseline_sorry[token]
        if delta > 0:
            issues.append(issue(path, "no_new_sorry_admit", "reject",
                                f"Introduced {delta} new `{token}` token(s)."))

    baseline_unsound = token_counts(baseline_text or "", ["axiom", "unsafe"])
    current_unsound = token_counts(current_text, ["axiom", "unsafe"])
    for token in ("axiom", "unsafe"):
        delta = current_unsound[token] - baseline_unsound[token]
        if delta > 0:
            issues.append(issue(path, "no_new_axiom_unsafe", "reject",
                                f"Introduced {delta} new `{token}` token(s)."))

    base_imports = parse_imports(baseline_text or "")
    current_imports = parse_imports(current_text)
    for added in sorted(current_imports - base_imports):
        issues.append(issue(path, "import_guard", "needs-human-review",
                            f"Added import requires review: `{added}`."))

    base_fp, _ = public_declaration_fingerprints(baseline_text or "")
    current_fp, current_docs = public_declaration_fingerprints(current_text)
    if not allow_public_api_changes:
        for key in sorted(set(base_fp) | set(current_fp)):
            if key not in base_fp:
                meta = current_fp[key]
                issues.append(issue(path, "public_api_fingerprint_guard", "reject",
                                    f"Added public declaration `{key}`.", meta["line"]))
            elif key not in current_fp:
                meta = base_fp[key]
                issues.append(issue(path, "public_api_fingerprint_guard", "reject",
                                    f"Removed public declaration `{key}`.", meta["line"]))
            elif base_fp[key]["header"] != current_fp[key]["header"]:
                meta = current_fp[key]
                issues.append(issue(path, "public_api_fingerprint_guard", "reject",
                                    f"Changed public declaration header `{key}`.", meta["line"],
                                    {"before": base_fp[key]["header"], "after": meta["header"]}))

    if "/-!" not in "\n".join(current_text.splitlines()[:40]):
        issues.append(issue(path, "mathlib_docstring_review", "needs-human-review",
                            "Touched file has no module docstring in the first 40 lines."))

    for idx, line in enumerate(current_text.splitlines(), start=1):
        if len(line) > 100:
            issues.append(issue(path, "mathlib_style_review", "needs-human-review",
                                "Line exceeds Mathlib's 100-character style target.", idx,
                                {"length": len(line)}))

    for key, meta in current_fp.items():
        kind = meta["kind"]
        name = meta["name"]
        if kind in {"theorem", "lemma", "def", "abbrev", "structure", "class", "inductive"}:
            if not current_docs.get(key):
                issues.append(issue(path, "mathlib_docstring_review", "needs-human-review",
                                    f"Public declaration `{key}` has no nearby docstring.",
                                    meta["line"]))
        if kind in {"theorem", "lemma"} and not is_snake_case(name):
            issues.append(issue(path, "mathlib_naming_review", "needs-human-review",
                                f"Theorem/lemma `{name}` is not snake_case.", meta["line"]))
        if kind in {"structure", "class", "inductive"} and not is_upper_camel(name):
            issues.append(issue(path, "mathlib_naming_review", "needs-human-review",
                                f"Type-like declaration `{name}` is not UpperCamelCase.",
                                meta["line"]))

    return issues


def write_markdown(report, path):
    lines = [
        "# Mathlib-Strict Gate Report",
        "",
        f"**Profile:** `{report['profile']}`",
        f"**Status:** `{report['status']}`",
        f"**Files:** {len(report['files'])}",
        f"**Issues:** {len(report['issues'])}",
        "",
    ]
    if report["issues"]:
        lines.append("## Issues")
        lines.append("")
        for item in report["issues"]:
            loc = item["file"]
            if "line" in item:
                loc = f"{loc}:{item['line']}"
            lines.append(f"- `{item['status']}` `{item['gate']}` {loc}: {item['message']}")
    else:
        lines.append("No static gate findings.")
    lines.append("")
    lines.append("This static gate complements Lean build, lint, test, and human review.")
    path.write_text("\n".join(lines) + "\n", encoding="utf-8")


def parse_args(argv):
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("targets", nargs="*", help="Lean files or directories to check")
    parser.add_argument("--profile", default=PROFILE, choices=[PROFILE])
    parser.add_argument("--baseline-dir", type=Path)
    parser.add_argument("--report-json", type=Path)
    parser.add_argument("--report-md", type=Path)
    parser.add_argument("--report-only", action="store_true")
    parser.add_argument("--fail-on-review", action="store_true")
    parser.add_argument("--allow-public-api-changes", action="store_true")
    return parser.parse_args(argv)


def main(argv=None):
    args = parse_args(argv or sys.argv[1:])
    files = expand_targets(args.targets) if args.targets else changed_lean_files_from_git()
    if not files:
        print("No Lean files to check.", file=sys.stderr)
        report = {
            "version": VERSION,
            "profile": args.profile,
            "status": "accept",
            "files": [],
            "issues": [],
            "summary": {"reject": 0, "needs-human-review": 0},
        }
    else:
        all_issues = []
        checked = []
        for path in files:
            rel = Path(path)
            baseline = read_baseline(rel, args.baseline_dir)
            checked.append(str(rel))
            all_issues.extend(analyze_file(rel, baseline, args.allow_public_api_changes))
        status = "accept"
        for item in all_issues:
            status = status_join(status, item["status"])
        report = {
            "version": VERSION,
            "profile": args.profile,
            "status": status,
            "files": checked,
            "issues": all_issues,
            "summary": {
                "reject": sum(1 for item in all_issues if item["status"] == "reject"),
                "needs-human-review": sum(1 for item in all_issues
                                          if item["status"] == "needs-human-review"),
            },
        }

    text = json.dumps(report, indent=2, sort_keys=True)
    if args.report_json:
        args.report_json.write_text(text + "\n", encoding="utf-8")
    else:
        print(text)
    if args.report_md:
        write_markdown(report, args.report_md)

    if args.report_only:
        return 0
    if report["status"] == "reject":
        return 1
    if report["status"] == "needs-human-review" and args.fail_on_review:
        return 1
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 2: Run the focused test**

Run:

```bash
python3 plugins/lean4/lib/scripts/tests/test_mathlib_strict_gate.py
```

Expected: PASS.

- [ ] **Step 3: Check direct CLI behavior manually**

Run:

```bash
tmpdir=$(mktemp -d)
mkdir -p "$tmpdir/base" "$tmpdir/repo"
printf '%s\n' 'import Mathlib' '' '/-! # Demo -/' '' '/-- T. -/' 'theorem t : True := by trivial' > "$tmpdir/base/Demo.lean"
printf '%s\n' 'import Mathlib' '' '/-! # Demo -/' '' '/-- T. -/' 'theorem t : True := by sorry' > "$tmpdir/repo/Demo.lean"
(cd "$tmpdir/repo" && python3 /Users/hoxide/mycodes/lean4-skills/plugins/lean4/lib/scripts/mathlib_strict_gate.py Demo.lean --baseline-dir "$tmpdir/base" --report-json "$tmpdir/report.json")
```

Expected: command exits `1`; `"$tmpdir/report.json"` contains `"status": "reject"` and `no_new_sorry_admit`.

### Task 3: Wire Documentation To The New Gate

**Files:**
- Modify: `plugins/lean4/lib/scripts/README.md`
- Modify: `plugins/lean4/lib/scripts/TESTING.md`
- Modify: `plugins/lean4/commands/review.md`
- Modify: `plugins/lean4/commands/refactor.md`
- Modify: `plugins/lean4/commands/golf.md`
- Modify: `plugins/lean4/skills/lean4/SKILL.md`
- Modify: `plugins/lean4/skills/lean4/references/mathlib-quality-system.md`

- [ ] **Step 1: Document the script in `plugins/lean4/lib/scripts/README.md`**

Add this row to the script overview table:

```markdown
| `mathlib_strict_gate.py` | Diff-aware Mathlib-strict static gate | Before accepting AI rewrite diffs |
```

Add this section after `check_axioms_inline.sh`:

```markdown
### mathlib_strict_gate.py

Diff-aware static gate for Mathlib-style rewrite safety. It compares touched
Lean files against a baseline and reports hard rejects (`sorry`, `admit`,
source-level `axiom`/`unsafe`, public API fingerprint drift) plus review items
(imports, missing docs, long lines, naming warnings).

```bash
./mathlib_strict_gate.py src/File.lean --baseline-dir /tmp/baseline --report-json report.json --report-md report.md
./mathlib_strict_gate.py --report-only
./mathlib_strict_gate.py --fail-on-review
```

Exit codes: `1` for `reject`, `0` for `accept` and `needs-human-review`.
Use `--fail-on-review` when CI should also fail controlled review findings.
This static gate complements `lake build`, `lake test`, `lake exe lint-style`,
`lake exe lintAll`, and human maintainer review.
```

Add `mathlib_strict_gate.py` to the grep-style exit code paragraph only if it is phrased carefully:

```markdown
- `mathlib_strict_gate.py` - exit 1 when hard status is `reject`
```

- [ ] **Step 2: Document script testing in `plugins/lean4/lib/scripts/TESTING.md`**

Add this row:

```markdown
| `mathlib_strict_gate.py` | ✅ Production Ready | Static diff-aware gate for Mathlib-strict AI rewrites |
```

Add this section:

```markdown
## mathlib_strict_gate.py Tests

Validates hard rejects, human-review findings, comment/string stripping, public
API fingerprint checks, and report-only behavior.

```bash
python3 plugins/lean4/lib/scripts/tests/test_mathlib_strict_gate.py
```
```

- [ ] **Step 3: Extend `/lean4:review` docs**

In `plugins/lean4/commands/review.md`, add an optional input row:

```markdown
| --mathlib-strict | No | Include static Mathlib-strict gate report for changed/touched Lean files |
```

Add this action after Axiom Check:

```markdown
4. **Mathlib-Strict Static Gate** (batch/changed/file scopes, when requested or when reviewing AI rewrite diffs) - `${LEAN4_PYTHON_BIN:-python3} "$LEAN4_SCRIPTS/mathlib_strict_gate.py" <target> --report-json <tmp>/mathlib-strict.json --report-md <tmp>/mathlib-strict.md --report-only`
```

Renumber later actions. Add this output section:

```markdown
### Mathlib-Strict Gate
- Status: accept | needs-human-review | reject
- Hard rejects: N
- Review items: N
```

- [ ] **Step 4: Extend mutating command docs**

In `plugins/lean4/commands/refactor.md`, add this final verification sentence under Actions step 6:

```markdown
For Mathlib-style cleanup or AI rewrite batches, also run `mathlib_strict_gate.py` on touched files and reject any hard `reject` status before reporting success.
```

In `plugins/lean4/commands/golf.md`, add this safety bullet:

```markdown
- For Mathlib-strict runs, static gate rejects public header/API drift, new sorries, new admits, and new source-level axioms/unsafe code after proof edits.
```

- [ ] **Step 5: Update core skill and reference**

In `plugins/lean4/skills/lean4/SKILL.md`, add `mathlib_strict_gate.py` to Core Primitives:

```markdown
| `mathlib_strict_gate.py` | Static Mathlib-strict diff gate | JSON/Markdown |
```

In the Quality Gate section, add:

```markdown
For Mathlib-style AI rewrites, run the static diff gate before claiming a rewrite is acceptable:

```bash
${LEAN4_PYTHON_BIN:-python3} "$LEAN4_SCRIPTS/mathlib_strict_gate.py" <changed-file> --report-json mathlib-strict.json --report-md mathlib-strict.md
```

This gate is not a substitute for `lake build`, lint commands, or human API review.
```

In `plugins/lean4/skills/lean4/references/mathlib-quality-system.md`, update "Minimal Version" to say v1 has a static diff-aware runner and still needs Lean/lint gates:

```markdown
Version 1 starts with a static diff-aware runner, `mathlib_strict_gate.py`, for:

- no new `sorry` or `admit`;
- no new source-level `axiom` or `unsafe`;
- public API fingerprint/header drift rejection;
- import/docstring/line-width/naming review findings;
- JSON and Markdown reports.

It deliberately does not replace `lake build`, `lake test`, Mathlib linters, or human maintainer review.
```

### Task 4: Run Repository Verification

**Files:**
- Test: all files touched above.

- [ ] **Step 1: Run the new unit test**

Run:

```bash
python3 plugins/lean4/lib/scripts/tests/test_mathlib_strict_gate.py
```

Expected: PASS.

- [ ] **Step 2: Run existing script tests**

Run:

```bash
python3 plugins/lean4/lib/scripts/tests/test_ordering.py
python3 plugins/lean4/lib/scripts/test_apply_exact_chains.py
```

Expected: PASS.

- [ ] **Step 3: Run command parser tests**

Run:

```bash
python3 -m unittest discover -s plugins/lean4/tests/command_args -p 'test_*.py'
```

Expected: PASS.

- [ ] **Step 4: Run shell regression tests**

Run:

```bash
bash plugins/lean4/tests/test_guardrails.sh
bash plugins/lean4/tests/test_validate_user_prompt.sh
bash plugins/lean4/tests/test_bash3_smoke.sh
bash plugins/lean4/tests/test_lint_bash_compat.sh
```

Expected: PASS.

- [ ] **Step 5: Run documentation checks**

Run:

```bash
bash plugins/lean4/tools/lint_docs.sh
bash plugins/lean4/tools/test_contracts.sh
```

Expected: PASS, or pre-existing warnings only. If new warnings appear, fix the docs in this branch.

- [ ] **Step 6: Inspect git diff**

Run:

```bash
git diff -- plugins/lean4/lib/scripts/mathlib_strict_gate.py \
  plugins/lean4/lib/scripts/tests/test_mathlib_strict_gate.py \
  plugins/lean4/lib/scripts/README.md \
  plugins/lean4/lib/scripts/TESTING.md \
  plugins/lean4/commands/review.md \
  plugins/lean4/commands/refactor.md \
  plugins/lean4/commands/golf.md \
  plugins/lean4/skills/lean4/SKILL.md \
  plugins/lean4/skills/lean4/references/mathlib-quality-system.md
```

Expected: diff is limited to the planned files; no unrelated changes are reverted.

### Task 5: Final Review Notes

**Files:**
- No additional edits unless verification finds issues.

- [ ] **Step 1: Confirm official-standard alignment**

Check final docs still distinguish:

- static script gates;
- Lean build/test/lint gates;
- human Mathlib maintainer review;
- proposal-only API/notation/instance/macro changes.

Expected: no document claims that the static gate alone proves Mathlib acceptance.

- [ ] **Step 2: Prepare completion summary**

Summarize:

- new script and behavior;
- tests run and results;
- any verification that could not run;
- remaining next step: optional integration into CI or a future `/lean4:quality` command.

Do not claim full Mathlib quality automation. Claim only v1 static gate support.

