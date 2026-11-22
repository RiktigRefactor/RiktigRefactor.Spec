# Commit messages

Every successful result includes a suggested commit message in [Arlo's Commit Notation](https://github.com/RefactoringCombos/ArlosCommitNotation).

A refactoring done by a trusted automated tool is a provable refactoring, so the risk level is `.` and the intention is `r`:

```text
. r Rename Order.Totl to Order.Total

Behavior-preserving rename across 7 files (14 references).

Refactor: rename csharp src/Order.cs:12:17 Totl -> Total
Refactor-Tool: riktig-cs 1.4.0
```

- The `Refactor:` trailer is machine-readable.
  Reviewers or CI can replay the refactoring on the parent commit and confirm the diff matches exactly (`riktig-cs verify-commit <sha>`).
- Batch results produce one message with one trailer per action.
- Tools warn when a working tree mixes a refactoring with other changes, and suggest committing them separately.
  A commit that mixes them is no longer `. r`.
- Projects that don't use Arlo's Commit Notation can turn off the prefix.
