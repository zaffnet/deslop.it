---
name: deslop
description: >-
  Scans a Python directory for AI-generated code bloat ("slop") across 6 categories:
  over-defensive code, premature abstractions, dead weight, verbose patterns,
  structural bloat, and documentation/logging noise. Uses weighted scoring
  (1.5x for structural categories) and systematic verification (caller counts,
  data tracing, parameter tax). Calculates slop density and offers to auto-fix.
  Trigger phrases: "deslop", "scan for slop", "reduce slop", "clean up slop",
  "slop check", "find slop", "remove slop from directory".
argument-hint: <directory>
allowed-tools: Bash(pre-commit *), Read, Glob, Grep, Write, Edit, Task, EnterPlanMode
---

# Deslop

Scan `$ARGUMENTS` (or the current working directory if omitted) for AI-generated code bloat, calculate a weighted slop density score, and offer to auto-fix.

## Step 1: Resolve Directory & Discover Files

### 1a. Resolve the target directory

If `$ARGUMENTS` is provided, treat it as a directory path (absolute or relative to cwd). Verify it exists. If not provided, use the current working directory. Store as `TARGET_DIR`.

### 1b. Discover files

Use `Glob` with these patterns rooted at `TARGET_DIR`:

**Include:**

- `**/*.py` — Python source files
- `**/*.yaml`, `**/*.yml`, `**/*.toml`, `**/*.json`, `**/*.cfg` — config files

**Exclude (skip entirely):**

- `**/evaluation/data/**`
- `**/*.lock`
- `**/*.pyc`
- `**/*.md`
- `**/few_shot_examples/**/*.json` and similar data-directory JSON
- `**/__pycache__/**`
- `**/outputs/**`
- `**/.venv/**`, `**/node_modules/**`
- Binary files

### 1c. Skip test files

Remove any file matching:

- Files inside directories named `tests/` or `test/` at any depth
- Files named `test_*.py` or `*_test.py` at any depth
- `conftest.py` files

**Note:** Test files are excluded from scanning but are still included as search targets during `Grep`-based caller-count verification in Step 2b.

### 1d. Count total lines

For each remaining file, count the number of non-empty lines. Sum to get `total_lines`.

Print: `Scanning N files (P Python, C config) in TARGET_DIR. Total non-empty lines: T.`

**Exit criteria:** Complete list of files to scan, with line counts collected.

## Step 2: Analyze Files for Slop

### 2a. Read and scan each file

Read each file in full using the `Read` tool. Scan the full file contents against the **AI Slop Pattern Taxonomy** below. Every non-empty line in every file is eligible for slop detection.

For each finding, record:

- **Finding ID**: `S1`, `S2`, `S3`, ... (sequential across all files)
- **Pattern name** (bold name from the taxonomy, e.g., "Unreachable guard")
- **Category** (one of the 6 categories below, or Config slop)
- **Weight**: `1.0x` (Cat 1, 4, 6) or `1.5x` (Cat 2, 3, 5) — Config excluded from scoring
- **File and line range** (from the `Read` output line numbers)
- **Offending code** (verbatim from the file)
- **Suggested replacement** (the idiomatic version)
- **Estimated lines saved** (lines that would be removed or shortened)

### 2b. Systematic Verification

Before finalizing any finding, apply the relevant verification technique(s). A finding that fails verification must be discarded.

**Caller count verification** — Required for: one-caller wrappers (Cat 2), zero-caller code (Cat 3), delegation chains (Cat 5).
Use `Grep` to search the entire project (including test files) for call sites of the function/class. Count unique callers (not just matches — deduplicate by call site). A function called from 2+ distinct locations is NOT a one-caller wrapper. A function called from tests only may still be dead weight if it has no production callers.

**Data tracing** — Required for: unused parameters (Cat 3), configuration mirage (Cat 2).
Trace each parameter from its declaration through the function body. Does the parameter's value influence the return value or a side effect? If yes, it is not dead weight. For "configuration" constants, check whether any code path reads the value dynamically (not just at import time).

**Parameter tax analysis** — Required for: intermediate data structures (Cat 2), pre-computed state (Cat 3).
Count how many lines exist solely to build, pass, or unpack the intermediate structure. If removing the structure would eliminate more lines than it creates, it is dead weight. If the structure is used in 2+ distinct contexts, it earns its keep.

**Call chain depth mapping** — Required for: delegation chains (Cat 5).
Map the call chain from the public API entry point to the actual logic. If there are ≥2 intermediate functions that only delegate (add no logic, no error handling, no branching), the chain is a finding. If any intermediate function adds meaningful behavior, the chain is justified.

**Guard reachability check** — Required for: unreachable guards (Cat 1), redundant isinstance (Cat 1).
Check the type annotation and all callers. If every caller passes a non-None value and the type hint is non-Optional, the None guard is unreachable. If the type is `Any` or `Optional`, the guard is justified.

**Attribute access count** — Required for: wrapper classes (Cat 2), intermediate data structures (Cat 2).
Count how many unique attributes/methods of the wrapper are accessed by external code. If only 1-2 attributes are used, the wrapper is over-engineered. If 3+ attributes are meaningfully used, it may earn its keep.

### 2c. Parallelization for large directories

If there are more than 30 Python files, use the `Task` tool to parallelize analysis across file groups.

**Exit criteria:** All files analyzed. Findings list complete with IDs, pattern names, categories, weights, code snippets, and line-saved estimates. All findings have passed systematic verification.

### AI Slop Pattern Taxonomy

See `references/pattern-catalog.md` for full before/after code examples for every pattern.

#### Category 1: Over-Defensive Code (Weight: 1.0x)

- **Unreachable guard** — `if x is not None` on values that cannot be None (required params with non-Optional types)
- **Impossible except** — `try/except` around operations that cannot raise the caught exception
- **Redundant isinstance** — `isinstance()` on values whose type is statically known
- **Internal validation** — Input validation on internal-only functions (not at system boundaries)
- **Shadowed default** — Default parameter values that duplicate what every caller always provides
- **Guaranteed key** — Defensive `.get()` on dicts where the key is guaranteed to exist
- **Silent swallow** — `try/except` that catches exceptions and returns a default value, silently hiding errors that should propagate

#### Category 2: Premature Abstraction (Weight: 1.5x)

- **One-caller helper** — Helper function called exactly once from one call site
- **Transparent wrapper** — Wrapper class that adds no behavior, just delegates to the wrapped object
- **Configuration mirage** — Constants for values that are never configurable and never change
- **One-product factory** — Factory function that only ever creates one type of object
- **Single-variant generalization** — Parameterizing behavior that has only one variant
- **Lonely enum** — Enum class with a single member
- **Intermediate data structure addiction** — Creating a data class/dict to pass between two functions when direct parameters would suffice

#### Category 3: Dead Weight (Weight: 1.5x)

- **Unused parameter** — Function parameter that is accepted but never read in the function body
- **Once-used constant** — Module-level constant used in exactly one place (inline the value)
- **Zero-caller code** — Functions, methods, or classes with zero callers anywhere in the project
- **Pre-computed state** — Variables computed and stored but only read once immediately after
- **Import-only module** — Module that only re-exports imports from other modules without adding logic

#### Category 4: Verbose Patterns (Weight: 1.0x)

- **Boolean long-hand** — `if condition: return True else: return False` instead of `return condition`
- **Manual list build** — Loop appending to a list where a list comprehension would suffice
- **String concatenation chain** — Multi-line string concatenation instead of f-string or `str.join()`
- **Constructor over literal** — `dict()` / `list()` constructor instead of `{}` / `[]` literal
- **One-shot import** — Importing a module to access one attribute exactly once
- **Explicit length check** — `len(x) == 0` / `len(x) > 0` instead of truthiness check
- **Redundant else** — `else` after `return` / `raise` / `continue`
- **Deep nesting / early-return phobia** — Multiple levels of `if` nesting that could be flattened with guard clauses

#### Category 5: Structural Bloat (Weight: 1.5x)

- **Delegation chain** — A calls B calls C, where B adds no logic (just passes arguments through)
- **Symmetry addiction** — Parallel code structures maintained for "consistency" when only one path is used
- **Gratuitous generics** — Type parameters or Protocol classes used in exactly one concrete instantiation
- **Error message over-engineering** — Complex error formatting (f-strings, multiline messages, context dicts) for errors that are never caught or displayed to users
- **Repetitive structure** — Copy-pasted code blocks with minor variations that should be a loop or mapping

#### Category 6: Documentation & Logging Noise (Weight: 1.0x)

- **Restating comment** — Comment that restates what the code already says (not comments explaining *why*)
- **Signature-restating docstring** — Docstring that only repeats the function signature without adding useful information
- **Redundant logging** — Log statements that add no diagnostic value (entry/exit at every function, dumping full objects)
- **Empty docstring sections** — Docstring with empty or placeholder Args/Returns/Raises sections (e.g., "N/A", restating param names)

#### Config/YAML Slop (no weight — reported separately)

- Unnecessary formatting complexity or inconsistency without semantic purpose
- Commented-out code or settings in config files
- Redundant default values that match the tool's built-in defaults

## Step 3: Calculate Slop Score & Present Report

Calculate:

- `raw_slop_lines`: sum of estimated lines saved across all findings
- `weighted_slop_lines`: sum of `lines_saved * weight` for each finding (Config findings excluded)
- `total_lines`: sum of non-empty lines across all scanned files (from Step 1)
- `slop_density`: `(weighted_slop_lines / total_lines) * 100`

Interpret the score:

| Slop Density | Rating |
|---|---|
| 0–5% | Excellent — minimal slop |
| 5–15% | Good — minor cleanup possible |
| 15–30% | Needs work — significant bloat |
| 30%+ | Heavy slop — major cleanup needed |

### Zero-Slop Fast Path

If no findings, output:

```
Directory looks clean. N files scanned (T total lines), no significant slop detected.
```

And stop. Do not proceed to Steps 4–5.

### Report Format

```
## Slop Analysis: <directory-path>

### Score
- **Slop density: XX%** (<rating>) — weighted
- Files scanned: N (T total non-empty lines)
- Raw slop lines: R | Weighted slop lines: W
- Potential reduction: T → T-R lines (YY% smaller)

### Findings (ranked by weighted lines saved, highest first)

**S1** — **<Pattern Name>** — Cat <N> (<weight>) — `file.py:10-25` — **~15 lines saved**
> <description of the slop>
```python
# Current (slop)
<offending code>
```

```python
# Suggested (idiomatic)
<replacement code>
```

**S2** — ...

### Findings by Category

| Category | Weight | Findings | Raw Lines | Weighted Lines |
|----------|--------|----------|-----------|----------------|
| 1. Over-defensive code | 1.0x | 2 | 15 | 15.0 |
| 2. Premature abstraction | 1.5x | 1 | 8 | 12.0 |
| 3. Dead weight | 1.5x | 1 | 10 | 15.0 |
| 4. Verbose patterns | 1.0x | 4 | 12 | 12.0 |
| 5. Structural bloat | 1.5x | 0 | 0 | 0.0 |
| 6. Documentation noise | 1.0x | 1 | 3 | 3.0 |
| **Subtotal** | | **9** | **48** | **57.0** |
| Config slop (unweighted) | — | 1 | 3 | — |

```

## Step 4: Auto-Fix (Plan Mode)

After presenting the report, call `EnterPlanMode` with a plan that includes:

1. The full findings table from Step 3
2. A changes list: file, line range, what to change, finding ID, weight
3. Execution order: by file (minimize file opens), within each file by line number **descending** (avoid offset drift from earlier edits shifting later line numbers)
4. Apply Category 6 (documentation noise) fixes last — they are lowest risk and lowest impact
5. Verification plan: the project's verification suite (e.g., `pre-commit run --all-files`)

**Do not proceed until the user explicitly approves the plan.**

After plan approval, execute:

1. **Read** each file before editing (mandatory)
2. **Apply fixes** using the Edit tool, working from bottom to top within each file
3. **Run pre-commit**: `pre-commit run --all-files` (or project equivalent)
4. If pre-commit fails, fix lint/format/type issues and re-run (max 3 iterations)
5. If pre-commit still fails after 3 iterations, stop and report the failures

**Exit criteria:** All approved fixes applied. Pre-commit passes (or remaining failures reported after 3 iterations).

## Step 5: Report Results

```

## Slop Cleanup Complete

### Before → After

- Total non-empty lines: N → M (reduced by K, YY% smaller)
- Slop density: XX% → ZZ% (weighted)

### Changes Applied

| # | File | Finding | Weight | Lines Saved |
|---|------|---------|--------|-------------|
| 1 | foo.py | S1: Removed unreachable guard | 1.0x | -12 |
| 2 | bar.py | S3: Inlined one-caller helper | 1.5x | -8 |

### Verification

- Pre-commit: PASS / FAIL

### Next Steps

- Review changes and commit when ready

```

Never auto-commit. Leave that to the user.

## Important Guidelines

- **Full-file scanning** — Scan every non-empty line in every eligible file. All slop in the scanned directory is in scope, regardless of when it was introduced.
- **Works without git** — This skill does not require a git repository. It operates on filesystem paths only. Never invoke `git` or `gh` commands.
- **Documentation noise IS in scope** — Category 6 covers comments that restate the code, docstrings that just repeat the function signature, redundant logging, and empty docstring sections. However, useful documentation (explains *why*, documents non-obvious behavior, describes complex algorithms) is NOT slop. When in doubt, leave it.
- **Caller count is mandatory** — For Categories 2, 3, and 5, you MUST run `Grep` to verify caller counts before finalizing any finding. A finding without caller-count verification is invalid. Search the entire project including test files.
- **Weighted scoring** — Categories 2 (Premature Abstraction), 3 (Dead Weight), and 5 (Structural Bloat) carry 1.5x weight because structural issues are harder to detect and cause more long-term harm than surface-level verbosity.
- **Delete-then-reason test** — For any finding you're uncertain about: mentally delete the code and ask "what breaks?" If nothing breaks and no information is lost, it's slop. If something breaks or the code serves a documented purpose, it's not.
- **Structural awareness** — For Categories 2, 3, and 5 (the 1.5x-weighted categories), every finding must include a verification note explaining what technique was used and what the result was. Findings without verification notes are invalid.
- **Absolute standard** — Judge against ideal Python idioms, not surrounding codebase style. The goal is to write the best possible Python, not to match existing patterns that may themselves be sloppy.
- **Preserve functionality** — Every suggested change must be behavior-preserving. Never remove error handling at system boundaries.
- **Python + config** — Analyze `.py` files for code slop; flag noise in config files.
- **Read before editing** — Always read the full file before making any changes (during the fix phase).
- **Never fabricate findings** — Only flag patterns you can verify from the actual file contents. If you're unsure whether something is slop, leave it alone.
- **Respect system boundaries** — Input validation on user input, external APIs, and public interfaces is NOT slop.
- **Test files excluded from scanning** — Test files are excluded at discovery time (Step 1) and are never scanned. However, test files ARE included in `Grep` searches during verification (Step 2b) to get accurate caller counts.
- **Minimal changes** — Only change code that matches a slop pattern. Don't "improve" unrelated code.
- **Bottom-to-top editing** — When applying fixes within a file, work from the highest line number to the lowest to prevent offset drift.
