# To do

Everything that still needs to be considered or addressed, across all repos.

## Decisions needed

- [ ] Add the C# refusal codes `dynamic-access-possible`, `string-names-symbol`, and `would-introduce-errors` to the spec, or rename them to match existing spec codes.
- [ ] Align the test-case format between the spec and the C# repo.
  The spec writes a refusal as `"expect": {"refuse": ...}`; the C# tests use a top-level `refuse` field and a `naiveShouldFail` flag the spec doesn't describe.
  Recommendation: update the spec to match the 51 existing cases.
- [ ] Decide whether a rename that changes a `[CallerMemberName]` or `[CallerArgumentExpression]` value is a behavior change (refuse or warn) or intended, like `nameof`.
- [ ] Decide whether JSON results use workspace-relative or absolute file paths.
- [ ] Decide whether Roslyn's cast-inserting conflict repairs (e.g. `G(1)` → `F((short)1)`) should ever be accepted, or always refused.
- [ ] Decide how to cite language specifications in tests (which edition and numbering).
- [ ] Decide whether release notes must list new refusal codes, since new refusals aren't breaking changes.
- [ ] Decide how each release's next version is chosen (manual, or derived from commit notation).
- [ ] Decide whether to add a Dependabot auto-merge workflow, as the template repo has.

## Pending pull requests

- [ ] RiktigRefactor.CSharp #2: bump .NET SDK from 10.0.115 to 11.0.100 (CI passed).
- [ ] RiktigRefactor.CSharp #1 and RiktigRefactor.Spec #1: bump the Markdown link-check action.

## C#: correctness review

The first independent review was cut short by a session restart.
Still unchecked:

- [ ] Solutions with more than one project.
- [ ] Generated code and source generators.
- [ ] Partial methods, primary constructors, extension blocks, top-level statements.
- [ ] Names in attributes, crefs, labels, and aliases.
- [ ] Tuple element names and anonymous-type member names inferred from a renamed member.
- [ ] `this.` qualification inserted in static contexts.
- [ ] Whether every existing edge-case assertion really depends on its edge case.
- [ ] The CLI, scripts, and workflows.
- [ ] Regression tests for each implicitly looked-up name (`GetEnumerator`, `Dispose`, `GetAwaiter`, `Deconstruct`, query methods, `Length`/`Count`/`Slice`, interpolated-string handlers); only `Add` has one.
- [ ] Verify the C# specification section numbers cited in test cases; they were cited from memory.

## C#: known false refusals and precision

- [ ] Renaming a member of an anonymous type is refused (safe, but unnecessary).
- [ ] The implicit-name check refuses based on syntax kinds anywhere in the workspace.
  A precise version would compare what those constructs bind to before and after the rename, in `BindingVerifier`.

## C#: not yet implemented

- [ ] Extract IIFE.
- [ ] Batch mode (`batch`, JSONL, snapshot semantics, conflict detection, combined re-binding check).
- [ ] MCP server (`riktig-cs mcp`).
- [ ] Language server, for the VS Code extension.
- [ ] Commit message generation (`--commit-message`, the protocol `commit` object, `verify-commit`).
- [ ] `--range` selection and JSON request input in the CLI.
- [ ] `explain_refusal`.
- [ ] Skill (`SKILL.md`) and `llms.txt` for AI agents.
- [ ] NuGet publishing workflow (needs an API key secret), and reserving the `RiktigRefactor.*` prefix on NuGet.
- [ ] Opt-in telemetry and AI-assisted issue reporting.

## Testing infrastructure

- [ ] Fuzzing: pick open-source corpus projects, random selections and names, nightly job, reduce failures to test cases.
- [ ] Differential execution against corpus projects' own test suites.
- [ ] Spec-coverage report: which language-spec sections have no tests.
- [ ] Batch-mode test cases (overlap, collision, swap, nested selection, order independence).
- [ ] Define the fuzzing threshold for promoting a release.

## Other languages and repos

- [ ] RiktigRefactor.Python: choose libraries (e.g. LibCST plus a scope analyzer), implement Rename and Extract IIFE.
- [ ] RiktigRefactor.Cpp: clang LibTooling, pin a clang version, require `compile_commands.json`.
- [ ] RiktigRefactor.VSCode: thin client over each language server; code actions, disabled actions with reasons, refactor preview, "Riktig Rename" command.
- [ ] Allow network access for PyPI and LLVM downloads in cloud sessions before starting Python and C++.

## Planned features (documented as limitations)

- [ ] Support older language versions, for recovering legacy code.
- [ ] Conditional compilation (`#if`, C++ preprocessor), on request.
- [ ] Refactorings that remove `return`/`break`/`continue`/`await` from a selection, so Extract IIFE can proceed.

## Process and tooling

- [ ] Write a skill for the test-case format, including regenerating expected output with `RIKTIG_WRITE_EXPECTED=1` and reviewing it.
- [ ] Keep the shared skills in sync across repos (submodule or sync script) instead of copies.
- [ ] Update the arlos-commit-notation skill to list this project's extension intentions and dependency-update prefixes.
- [ ] Fix `- r` to `^ r` in the template repo's Dependabot config.
- [ ] Add an "adopt conventions" skill or script for setting up new repos.
- [ ] Document the network hosts cloud sessions need (NuGet, PyPI, LLVM).
