# Commit messages

Every successful result includes a suggested commit message:

```
Rename Order.Totl to Order.Total

Pure refactoring: behavior-preserving rename across 7 files
(14 references). No other changes.

Refactor: rename csharp src/Order.cs:12:17 Totl -> Total
Refactor-Tool: riktig-cs 42
```

- The `Refactor:` trailer is machine-readable. Reviewers or CI can replay the refactoring on the parent commit and confirm the diff matches exactly (`riktig-cs verify-commit <sha>`).
- Batch results produce one message with one trailer per action.
- Tools warn when a working tree mixes a refactoring with other changes, and suggest committing them separately.
