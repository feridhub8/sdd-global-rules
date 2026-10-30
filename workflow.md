# Workflow: Spec-Driven Development

- Every project uses OpenSpec. The AI tool is chosen per project: `openspec init --tools <tool>`. Ask me which one.
- Spec before code: do not create service folders or write code before there is a proposal for it.
- Flow: explore (think it through) → propose (proposal, specs, design, tasks) → apply (implement) → archive (merge into specs).
- On archive, always sync the change's delta specs into the main specs; do not ask whether to sync. Stop and ask only when a capability cannot be synced (for example, a MODIFIED requirement with no main spec).
- Keep each change small: one service or one feature per proposal.
- Constraints the assistant cannot infer from the code (naming, stack, repo layout, database decisions, scope) go in the project's `openspec/config.yaml` under `context`.
