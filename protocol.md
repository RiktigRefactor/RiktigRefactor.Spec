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
  "workspaceRoot": "/home/ana/shop",
  "edits": [ { "file": "src/Foo.cs", "start": {...}, "end": {...}, "newText": "count" } ],
  "warnings": [ { "code": "trivia-moved", "message": "...", "location": {...} } ],
  "commit": { "subject": "...", "body": "...", "trailer": "Refactor: ..." }
}
```

Refusal:

```json
{
  "status": "refused",
  "workspaceRoot": "/home/ana/shop",
  "reasons": [ {
    "code": "non-local-control-flow",
    "message": "The selection contains `continue`, which would have no enclosing loop inside the lambda.",
    "location": { ... },
    "suggestion": "Shrink the selection to exclude line 42."
  } ]
}
```

Every `file` in a result is relative to `workspaceRoot` and uses `/` as the separator on every OS, so results are the same on every machine.
`workspaceRoot` is absolute, in the OS's native form; clients that need absolute paths join it with `file`.

Reason codes are stable.
Messages may change.
New codes may be added in any minor version, so clients must handle codes they don't recognize (see [Releases](process.md#releases)).

Each language defines its own reason codes, because the hazards differ by language.
The spec does not list or require any code other than `internal-verification-failed`; codes in examples, such as the one above, are illustrative.
Each language repo is the source of truth for its codes, and documents them where agents and `explain_refusal` can find them.
Only `internal-verification-failed` is common to all languages (see [principles.md](principles.md)).

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
