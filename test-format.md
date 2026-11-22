# Test format

Every test is executable.
Both the input and the expected output must compile (where applicable), run, and pass an assertion.
This guarantees:

- (A) the code being refactored is legal and does what the test author thinks it does;
- (B) the refactored code still does the same thing.

## Layout

Each test case is a directory, so cross-file refactorings are first-class:

```
tests/rename/shadowing/inner-capture/
  case.json          # refactoring, options, expected outcome, spec citations
  input/             # one or more source files, with selection markers
  expected/          # expected output files (omitted when a refusal is expected)
```

## case.json

```json
{
  "refactoring": "rename",
  "options": { "newName": "count" },
  "expect": "success",
  "warnings": [],
  "spec": ["C# 13 §7.7.1"],
  "notes": "Inner reference would be captured by an existing local."
}
```

For an expected refusal: `"expect": { "refuse": "binding-would-change" }` and no `expected/` directory.

Every test cites the language-specification sections it covers, so we can report which sections lack tests.

## Selection markers

### C# and C++ (and other languages with block comments)

- Range: `/*[*/` … `/*]*/`
- Cursor: `/*|*/`

Markers are ordinary comments, so files remain valid with markers in place.

### Python (and other languages with line comments only)

Single-line selection: a trailing comment that quotes the selected text.

```python
total = price * qty + shipping  # [price * qty]
total = x + x                    # [x]#2      <- second occurrence
print(count)                     # |count     <- cursor at start of "count"
```

Multi-line selection: whole lines between marker lines.

```python
# [
a = f(x)
b = g(a)
# ]
```

Multi-line selections start and end on line boundaries.

### Handling

The runner removes markers before invoking the refactoring.
Expected output contains no markers.
Only one selection per test file (batch tests use a JSONL file instead; see below).

## Assertions

- Every input contains at least one assertion, checked when the program runs.
- For order-sensitive behavior, test code appends to a trace log and asserts on the whole trace.
  This catches double evaluation and reordering.
- **Assertions must be able to fail.** Each refactoring keeps a deliberately naive implementation.
  CI checks that every edge-case test fails under the naive version (cheap mutation testing).
  A test that passes under the naive version is flagged.

## What the runner checks

1. The input runs and passes its assertion.
   (Failure = bad test.)
1. The refactoring produces the expected outcome (success or the specified refusal code).
1. Output matches `expected/` exactly, trivia included.
   (Mismatch with a passing step 4 = cosmetic issue.)
1. The output runs and passes its assertion.
   (Failure = correctness bug, highest severity.)

## Batch tests

```
tests/batch/swap/
  case.json
  actions.jsonl      # one request per line, locations relative to input/
  input/
  expected/
```

Required batch cases: textual overlap, name collision (`a→c`, `b→c`), swap (`a→b`, `b→a`), IIFE containing a renamed reference.
Property: for batches with no interactions, any ordering of lines gives the same result.

## Sources of tests

- Hand-written and model-generated edge cases, guided by a systematic review of each language specification.
  No human review is required for model-generated tests; the executable assertion plus the naive-implementation mutation check is the bar.
- Mechanical fuzzing over open-source GitHub code (see [process.md](process.md)).
