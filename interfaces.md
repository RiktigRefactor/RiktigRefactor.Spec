# Interfaces

The **library** is the implementation.
The CLI, MCP server, and language server are thin wrappers that behave identically.
The test suite runs through the library; a small set of integration tests covers each wrapper.

## Naming

Pattern: `RiktigRefactor.{Language}`, matching the repo names, adapted to each ecosystem's conventions.

| Language | Library | Tool (CLI + MCP) |
|---|---|---|
| C# | NuGet `RiktigRefactor.CSharp` | dotnet tool `RiktigRefactor.CSharp.Tool`, command `riktig-cs` |
| Python | PyPI `riktigrefactor-python`, import `riktigrefactor` | same package, command `riktig-py` |
| C++ | `riktigrefactor-cpp` (vcpkg, Conan) | binary `riktig-cpp` (Homebrew, release binaries) |
| VS Code | extension `riktigrefactor.vscode` | — |

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
  Tools: one per refactoring (`rename`, `extract_iife`, `move_static_member`, `make_method_static`, `move_to_instance_method`), `batch`, `explain_refusal`, `list_refactorings`.
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
