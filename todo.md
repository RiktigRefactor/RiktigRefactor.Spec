# To do

Everything that still needs to be considered or addressed, across all repos.

## Decisions needed

- [ ] Decide whether Roslyn's cast-inserting conflict repairs (e.g. `G(1)` → `F((short)1)`) should ever be accepted, or always refused.
- [ ] Decide how to cite language specifications in tests (which edition and numbering).
- [ ] Decide whether release notes must list new refusal codes, since new refusals aren't breaking changes.
- [ ] Decide how each release's next version is chosen (manual, or derived from commit notation).
- [ ] Decide whether to add a Dependabot auto-merge workflow, as the template repo has.
- [ ] Decide whether renaming an enum member is a behavior change: it changes `ToString()` and `Enum.Parse` results.
  The `basic/enum-member` case currently accepts it.
  This is the same kind of issue as records, whose renames are now refused because `ToString` prints member names.
- [ ] Confirm the change to `basic/record-positional-parameter`: it used to expect success and now expects refusal (`string-names-symbol`).

## Pending pull requests

- [ ] RiktigRefactor.CSharp #2: bump .NET SDK from 10.0.115 to 11.0.100 (CI passed).
- [ ] RiktigRefactor.Spec #1: bump the Markdown link-check action (the C# repo's copy is merged).

## C#: correctness review

The second review covered multi-project solutions, generated code, newer language features, attributes/crefs/labels/aliases, inferred tuple and anonymous-type names, static contexts, implicitly looked-up names, existing edge-case assertions, and the CLI.
It fixed 8 bugs; the suite has 96 tests.
Still unchecked:

- [ ] `[CollectionBuilder]` capture by overload (only lightly probed).
- [ ] Case-insensitive path comparison on Windows in the CLI.
- [ ] CI workflows and scripts, beyond confirming they pass.
- [ ] Verify the C# specification section numbers cited in test cases; they were cited from memory.
- [ ] A test for the CLI's exit code 2 on unexpected exceptions (no triggering input found yet).

## C#: known false refusals and precision

- [ ] Renaming a member of an anonymous type is refused (safe, but unnecessary).
- [ ] A named attribute argument `[A(Prop = 1)]` blocks renaming `Prop`, and renaming an attribute class used as `[A]` is refused, probably because of how the verifier identifies the attribute's implicit constructor.
- [ ] Calling a record's generated `Deconstruct` explicitly blocks renaming its properties.
- [ ] Every instance member of a record is refused, including private ones that `ToString` doesn't print.
- [ ] Any record in the workspace blocks renames to or from `Equals`, `GetHashCode`, `ToString`, `PrintMembers`, or `Deconstruct`.
- [ ] Any generated file containing the old or new name blocks the rename, even when the name means something unrelated there.
- [ ] The implicit-name check refuses based on syntax kinds anywhere in the workspace.
  A precise version would compare what those constructs bind to before and after the rename, in `BindingVerifier`.

## C#: not yet implemented

- [ ] Report workspace-relative paths with `/` and a top-level `workspaceRoot` in results (see protocol.md); results currently use Roslyn's absolute `FilePath`.
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

- [ ] Write a skill for the test-case format, including regenerating expected output with `RIKTIG_WRITE_EXPECTED=1` and reviewing it, and scaffolding new cases.
  Note that the case format can't express multiple projects or source generators; those tests live in `RenameWorkspaceTests.cs`.
- [ ] Make `RIKTIG_WRITE_EXPECTED=1` work in one run (it currently needs two).
- [ ] Document how to run a single test case (the full suite takes about 50 seconds).
- [ ] Keep the shared skills in sync across repos (submodule or sync script) instead of copies.
- [ ] Update the arlos-commit-notation skill to list this project's extension intentions and dependency-update prefixes.
- [ ] Fix `- r` to `^ r` in the template repo's Dependabot config.
- [ ] Add an "adopt conventions" skill or script for setting up new repos.
- [ ] Document the network hosts cloud sessions need (NuGet, PyPI, LLVM).
