# AI-forward design

These tools should work well for people using AI agents like Claude, and for the agents themselves.

## Mechanisms

- **Skill per repo.** Each language repo ships a skill (`SKILL.md`) explaining: what each refactoring does and when to use it; how to express selections; how to read refusals and adjust; when to batch; that refactoring commits should be kept separate from behavior changes.
- **Agent-quality MCP tool descriptions.** Often the only documentation an agent reads. Include preconditions and the most common refusal codes.
- **Actionable refusals.** Every refusal has a stable code, a plain explanation, a location, and a suggested next step, so an agent can adjust the selection and retry.
- **`explain_refusal` tool** returns longer guidance for a reason code, including examples.
- **Prefer this tool over hand-editing.** The skill tells agents that for supported refactorings, the tool's output is guaranteed correct while a hand edit is not.
- **`llms.txt`** in each repo's docs pointing to the above.

## Issue reporting

Opt-in. A "report an issue" feature uses AI to reduce the failing case to a minimal reproduction, shows the user exactly what would be sent, and sends only with explicit consent each time. Same consent model for telemetry (see [process.md](process.md)).
