# Interfaces

The **library** is the implementation.
The CLI, MCP server, and language server are thin wrappers that behave identically.
The test suite runs through the library; a small set of integration tests covers each wrapper.

## Naming

Pattern: `RiktigRefactor.{Language}`, matching the repo names, adapted to each ecosystem's conventions.

| Language | Library | Tool (CLI + MCP) |
|---|---|---|
| C# | NuGet `RiktigRefactor` | dotnet tool `RiktigRefactor.CSharp.Tool`, command `riktig-cs` |
| Python | PyPI `riktigrefactor`, import `riktigrefactor` | same package, command `riktig-py` |
| C++ | `riktigrefactor` (vcpkg, Conan) | binary `riktig-cpp` (Homebrew, release binaries) |
| Java | [TBD] | riktig-java |

Reserve the `RiktigRefactor.*` prefix on NuGet.

One tool per language provides both the CLI and the MCP server (`riktig-cs mcp`), so one install gives both.

## CLI

```
riktig-cs rename --workspace App.sln --file src/Foo.cs --at 12:5 --new-name count
riktig-cs extract-iife --workspace App.sln --file src/Foo.cs --range 12:5-14:20
riktig-cs batch --workspace App.sln actions.jsonl [--best-effort]
riktig-cs mcp
```

- Default is preview: prints a unified diff.
  `--apply` writes files.
- `--json` emits the protocol result.
- Selection via `--at` / `--range` or markers in the file.
- `--commit-message` prints a generated commit message (see [commits.md](commits.md)).

## MCP

- One MCP server per language tool.
  Tools: one per refactoring (`rename`, `extract_iife`, `move_static_member`, `make_method_static`, `convert_to_instance_method`, `extract_object_from_arguments`), `batch`, `explain_refusal`, `list_refactorings`.
- Returns edits by default; writing files requires an explicit `apply: true`.
- Tool descriptions are written for agents (see [ai.md](ai.md)).

## Language server and VS Code

- Each language tool runs as a long-lived language server so the workspace stays loaded.
- The `vscode` extension is a thin client.
- Refactorings appear as code actions (lightbulb, Ctrl+.).
  Refactorings that would refuse appear **disabled with the reason**, so the user sees why without trying.
- Edits go through VS Code's refactor preview.
  Warnings appear there.
- Each refactoring is one workspace edit, undone with one Ctrl+Z.
- Rename is a separate command, "Riktig Rename".
  A setting lets it take over F2; off by default.

### LSP mapping

Every language tool starts its language server with `<tool> lsp`, speaking LSP over stdio with `Content-Length` framing.
The server needs no open-document tracking: the client saves modified files before it runs a refactoring, and the tool reads the files from disk.

**Workspace.**
The server uses `initializationOptions.workspace` when the client gives it; otherwise it looks in the workspace folder for the language's workspace file (C#: a `.sln`, `.slnx` or `.csproj`; C++: `compile_commands.json`, in the folder or in `build/`).
When it finds none, it still starts, and every refactoring is refused with a reason saying which file is missing.

**Capabilities.**
`codeActionProvider` with `codeActionKinds: ["refactor.rewrite.riktig"]`, and `executeCommandProvider` with the command `riktig.refactor`.

**Code actions.**
`textDocument/codeAction` returns one `CodeAction` per refactoring that can apply at the range, with the kind `refactor.rewrite.riktig.<refactoring>` (for example `refactor.rewrite.riktig.rename`).
Its `command` is a *client-side* command named `riktigrefactor.<refactoring>` whose arguments are the document URI and the LSP range, because only the client can prompt for options.
A refactoring that would refuse should come back with `disabled: { reason }`.
Servers may leave this out for a refactoring whose refusal depends on an option the user has not chosen yet (such as Rename's new name); the refusal then comes back when the user runs it.

**Running a refactoring.**
The client prompts for the options, converts the LSP range to the protocol's 1-based scalar-value positions, and sends `workspace/executeCommand` with command `riktig.refactor` and one argument: a protocol [request](protocol.md#request).
`location.file` may be absolute or relative to the workspace root.
The response is the protocol [result](protocol.md#result), unchanged.
The server does not send `workspace/applyEdit`; the client converts the result's edits (reading each file to turn scalar-value columns into UTF-16 offsets) and applies them as one `WorkspaceEdit` through the refactor preview, with each warning as a change annotation that needs confirmation.
A refusal is shown to the user with its reason, location and suggestion.
