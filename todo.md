# To do

Everything that still needs to be considered or addressed, across all repos.

## Decisions needed

- [ ] Add the C# refusal codes `dynamic-access-possible`, `string-names-symbol`, and `would-introduce-errors` to the spec, or rename them to match existing spec codes.
- [ ] Align the test-case format between the spec and the C# repo.
  The spec writes a refusal as `"expect": {"refuse": ...}`; the C# tests use a top-level `refuse` field and a `naiveShouldFail` flag the spec doesn't describe.
  Recommendation: update the spec to match the existing cases.
- [ ] Decide whether a rename that changes a `[CallerMemberName]` or `[CallerArgumentExpression]` value is a behavior change (refuse or warn) or intended, like `nameof`.
- [ ] Decide whether JSON results use workspace-relative or absolute file paths.
- [ ] Decide whether Roslyn's cast-inserting conflict repairs (e.g. `G(1)` → `F((short)1)`) should ever be accepted, or always refused.
- [ ] Decide how to cite language specifications in tests (which edition and numbering).
- [ ] Decide whether release notes must list new refusal codes, since new refusals aren't breaking changes.
- [ ] Decide how each release's next version is chosen (manual, or derived from commit notation).
- [ ] Decide whether the other repos (Spec, VSCode, CPP, Python) get the Dependabot auto-merge workflow. The C# repo has it (CSharp #7).
- [ ] Spec limitations.md now mentions enum member names twice: the new bullet under "Reflection and dynamic access" (Spec #13) and an older sentence under "Rename: names captured as strings change". Keep both, or drop one.

## Spec changes agreed

None open.

Done 2026-11-22: added enum member names, `typeof(T).Name`, and expression-tree member names to the spec's limitations as reflection (Spec #13, merged); described in the rename spec that a rename may write an implicit name explicitly to keep behavior (Spec #15, merged).

## Pending pull requests and branches

- [ ] RiktigRefactor.Spec #2 (replace markdown-link-check with lychee): its status couldn't be checked on 2026-11-22; confirm it is merged. Once it is, close the link-check bump PR Spec #1.
- [ ] Switch the template repo and other repos (VSCode, CPP, Python) to lychee too.
- [ ] Delete the leftover `lychee` branch in RiktigRefactor.CSharp (its PR #3 is merged; cloud sessions can't delete branches).
- [ ] Delete the leftover branches `limitations-reflection-names` (Spec #13), `todo-sync` (Spec #14), `rename-implicit-names` (Spec #15), and `todo-spec-15` in RiktigRefactor.Spec (merged; cloud sessions can't delete branches).

Done 2026-11-22: CSharp #1 (link-check bump), #2 (.NET SDK 11.0.100), #3 (lychee), #5, #6, #7 (Dependabot auto-merge) are merged; #4 (link-check retries) was closed unmerged; the branches `correctness-review-anonymous-types` and `dllimport-entry-point` are gone.

## C#: correctness review

Second pass (2026-11-22) probed each area with throwaway cases.
Found and fixed wrong output: renaming `Count` in `new { box.Count }` changed the anonymous type's ToString and equality when it was used through `object` (#9); renaming a `[DllImport]` method changed the native entry point (#10).
Found and fixed a broad false refusal: any rename in a file holding an anonymous type in `var` was refused (#9).
Checked, no wrong output: crefs, labels, aliases, attribute named arguments, attribute-suffix capture, partial properties, primary constructors, top-level statements, tuple element names (changes become compile errors), extension methods vs instance methods, overload capture, implicitly looked-up names (covered by cases), LINQ `select new { b.Count }`, `InterpolatedStringHandlerArgument` parameter names (become compile errors).

Still unchecked:

- [ ] Solutions with more than one project. The case harness (`TestWorkspace`) only builds a single project, so this needs a harness change first.
- [ ] Generated code and source generators (beyond the existing refusals).
- [ ] Whether every existing edge-case assertion really depends on its edge case.
- [ ] The CLI, scripts, and workflows.
- [ ] Verify the C# specification section numbers cited in test cases; they were cited from memory.

## C#: finding non-obvious cases systematically

The wrong-output bugs had the same shape: a name's spelling matters to the program, not just what it binds to. The verifier only checks what names bind to, so it can't see these. Instead of probing by intuition:

- [ ] Build a catalog of every construct where a name's text affects behavior, from two sources: the C# spec (search for "inferred", "name of", "simple name", string-valued attributes) and Roslyn's compiler (where lowering and emit read `Symbol.Name`: anonymous types, tuples, records, `DllImport`, caller-info, attributes). Each entry gets a case or a documented limitation. Keep the catalog in the repo.
- [ ] Widen the verifier from names to meaning: compare the `IOperation` trees of every changed member before and after (types, conversions, invoked members, anonymous type shapes), treating only the intended renames as equal. This would have caught the anonymous-type bug without anyone thinking of it.
- [ ] Add mutation checks: disable each verifier rule and precondition in turn and confirm some case fails. A rule no case depends on is untested or dead.
- [ ] Prioritize fuzzing (below), since it finds cases nobody thought of.

## C#: test harness

- [ ] A naive-rename program that overflows the stack kills the whole test host (seen with a renamed expression-bodied property). Run case programs in a separate process, or with a thread of bounded stack size.

## C#: known false refusals and precision

- [ ] Renaming a member of an anonymous type is refused (safe, but unnecessary). Re-check: #9 may have changed this.
- [ ] The implicit-name check refuses based on syntax kinds anywhere in the workspace.
  A precise version would compare what those constructs bind to before and after the rename, in `BindingVerifier`.
- [ ] Renaming a property used as a named attribute argument (`[My(Value = 5)]`) is refused: the verifier reports that `My` binds differently.
- [ ] Renaming a static field to a name used by a local or lambda parameter is refused instead of qualifying it with the type name.
- [ ] Renaming a positional record parameter is refused with a confusing message about `ToString` binding. Re-check: #8 (rename records) may have changed this.

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
- [ ] Document that cloud sessions need both the .NET 11 SDK and the .NET 10 runtime (tests target net10.0).
- [ ] Note in AGENTS.md that cloud sessions open PRs with `gh api repos/.../pulls`, since `gh pr create` (GraphQL) is blocked.
- [ ] Consider turning on "Automatically delete head branches" in the repo settings, since cloud sessions can't delete branches.
