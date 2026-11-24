# Process

## Language specification review

Each language repo maintains a checklist of language-spec sections relevant to each refactoring.
Tests cite sections (see [test-format.md](test-format.md)); CI reports sections with no tests.
Models systematically review the spec for edge cases and correctness, section by section.

## Fuzzing

- Corpus: open-source code from GitHub, with license recorded per project.
- Random inputs over the corpus for each refactoring: selections (Extract IIFE), targets and new names (Rename), members and destinations (Move static member), methods (Make method static), methods and receiver parameters (Convert static method to instance method), and methods and parameter groups (Extract object from arguments).
- Signature-changing refactorings are also checked by applying them followed by their inverse (Make method static, then Convert static method to instance method); the round trip must pass the same oracle and, where the inverse can restore the original text, reproduce it.
- Oracle: compile, re-binding check, and where the project has tests, run them before and after.
- Runs nightly, not on every commit.
- Any failure is reduced to a minimal test case and added to the suite.

## CI

GitHub Actions on every commit: build, full test suite, naive-implementation mutation check, spec coverage report.

## Releases

- Versions follow [Semantic Versioning](https://semver.org/).
  A breaking change is a major version; an additive feature is a minor version; a fix (including a new refusal code) is a patch.
- A new refusal code is not a breaking change: clients must handle codes they don't recognize.
  It is usually a patch version, and the release notes list each new code with a one-line meaning.
- Every commit to `main` produces a prerelease, versioned `<next version>-ci.<run number>`.
- `<next version>` is worked out from the [commit notation](CONTRIBUTING.md) since the last release:
  a breaking change means a major version, an additive feature (an uppercase `F` commit) means a minor version, any other uppercase commit (including one that adds a refusal code) means a patch, and lowercase commits alone mean no release.
  The AI proposes a major version only when it sees a breaking change, and the person approving the draft confirms or changes it.
- The AI creates the release, but only as a draft.
  It reads the commit history since the last release, works out the version, and writes the release notes, including every new refusal code with a one-line meaning.
  It flags commits whose prefix doesn't match the change, and anything that looks like an incompatible protocol change.
  A person reviews the draft, edits it if needed, and approves it; only then is it published.
- A release is promoted when it passes the full edge-case suite plus a fuzzing threshold on the corpus with zero correctness failures (threshold defined per language once fuzzing is running).
- Packaging uses each ecosystem's native tooling (NuGet/dotnet tool, PyPI, vcpkg/Conan/Homebrew, VS Code Marketplace).

## Telemetry

Opt-in only.
Refusal codes (not code) are the most useful signal for prioritizing future refactorings.
