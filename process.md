# Process

## Language specification review

Each language repo maintains a checklist of language-spec sections relevant to each refactoring.
Tests cite sections (see [test-format.md](test-format.md)); CI reports sections with no tests.
Models systematically review the spec for edge cases and correctness, section by section.

## Fuzzing

- Corpus: open-source code from GitHub, with license recorded per project.
- Random selections and random rename targets/new names over the corpus.
- Oracle: compile, re-binding check, and where the project has tests, run them before and after.
- Runs nightly, not on every commit.
- Any failure is reduced to a minimal test case and added to the suite.

## CI

GitHub Actions on every commit: build, full test suite, naive-implementation mutation check, spec coverage report.

## Releases

- Versions follow [Semantic Versioning](https://semver.org/).
  Only incompatible protocol changes are major versions.
- Every commit to `main` produces a prerelease, versioned `<next version>-ci.<run number>`.
- A release is promoted when it passes the full edge-case suite plus a fuzzing threshold on the corpus with zero correctness failures (threshold defined per language once fuzzing is running).
- Packaging uses each ecosystem's native tooling (NuGet/dotnet tool, PyPI, vcpkg/Conan/Homebrew, VS Code Marketplace).

## Telemetry

Opt-in only.
Refusal codes (not code) are the most useful signal for prioritizing future refactorings.
