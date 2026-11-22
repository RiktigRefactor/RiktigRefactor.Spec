# Protocol

One JSON schema is shared by every language and every interface (library, CLI, MCP, language server).

## Request

```json
{
  "refactoring": "rename",
  "workspace": "path/to/MyApp.sln",
  "location": { "file": "src/Foo.cs", "start": { "line": 12, "column": 5 }, "end": { "line": 12, "column": 5 } },
  "options": { "newName": "count" }
}
```

- Lines and columns are 1-based; columns count Unicode scalar values.
  A cursor has `start == end`.
- `location` may be omitted when the file contains selection markers (the CLI and test runner accept this).
- Workspace: C# `.sln`/`.csproj`; C++ `compile_commands.json` (required: without it, macros and includes cannot be resolved soundly, so the tool refuses); Python project root plus interpreter path.

## Result

Success:

```json
{
  "status": "success",
  "edits": [ { "file": "src/Foo.cs", "start": {...}, "end": {...}, "newText": "count" } ],
  "warnings": [ { "code": "trivia-moved", "message": "...", "location": {...} } ],
  "commit": { "subject": "...", "body": "...", "trailer": "Refactor: ..." }
}
```

Refusal:

```json
{
  "status": "refused",
  "reasons": [ {
    "code": "non-local-control-flow",
    "message": "The selection contains `continue`, which would have no enclosing loop inside the lambda.",
    "location": { ... },
    "suggestion": "Shrink the selection to exclude line 42."
  } ]
}
```

Reason codes are stable.
Messages may change.

## Batch mode

Input: JSON Lines, one request per line.
The workspace is loaded once.

**Snapshot semantics only.** Every location refers to the original files.
The tool maps each location through the edits of other actions.
There is no sequential mode.

### Conflicts

- **Textual overlap**: two actions edit overlapping ranges.
- **Semantic interaction**: edits that don't overlap still change each other's meaning, e.g.
  `a→c` and `b→c` in the same scope; an IIFE capturing a variable renamed by another action (must use the new name).
- **Swaps** (`a→b`, `b→a`) are valid under snapshot semantics and are supported.

Each action is checked against the snapshot, and then the **combined result** gets a final re-binding check.
Failures report which actions interact.

### Failure modes

- `--atomic` (default): if any action refuses or conflicts, nothing is applied.
- `--best-effort`: apply all actions that succeed without conflict; report the rest.

Output: one result per input line, in order, plus an overall status.
