# RiktigRefactor

*Riktig* (Swedish/Norwegian): correct, right, proper.

RiktigRefactor is a family of refactoring tools that **always preserve behavior**, including in weird edge cases, in untested code, and in very large and complicated code.
The bar: select an arbitrary chunk in the middle of a 10,000-line method, extract it, and ship to production without worry.

When a tool cannot prove a transformation is safe, it **refuses with a clear reason**.
A conservative tool that is always right beats a clever one that is right 99.9% of the time.

This repo holds the cross-language specification.
Each language has its own implementation repo.

## Repositories

| Repo | Contents | Implementation language |
|---|---|---|
| `RiktigRefactor.Spec` | This specification, shared schemas, shared test-format docs | — |
| `RiktigRefactor.CSharp` | C# library, CLI, MCP server, language server | C# on Roslyn |
| `RiktigRefactor.Python` | Python library, CLI, MCP server, language server | Python |
| `RiktigRefactor.Cpp` | C++ library, CLI, MCP server, language server | C++ on clang LibTooling |
| `RiktigRefactor.VSCode` | VS Code extension (thin client over each language server) | TypeScript |

Initial languages: C#, Python, C++.
The goal is to cover every language people care about.

## Contents

- [principles.md](principles.md) — correctness model, refusal policy
- [refactorings/rename.md](refactorings/rename.md)
- [refactorings/extract-iife.md](refactorings/extract-iife.md)
- [test-format.md](test-format.md) — executable input/expected test cases
- [protocol.md](protocol.md) — request/result schema, batch mode
- [interfaces.md](interfaces.md) — library, CLI, MCP, VS Code
- [ai.md](ai.md) — making the tools work well for AI agents
- [commits.md](commits.md) — commit message generation
- [limitations.md](limitations.md) — what is out of scope
- [process.md](process.md) — testing, fuzzing, releases, packaging, telemetry

## License

MIT.
