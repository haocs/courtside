---
name: courtside-change
description: Route, implement and validate work across the Courtside tennis component repositories. Use for Courtside feature or bug work and new-agent handoff; not unrelated projects or production deployment by itself.
---

# Courtside component change

Locate the hub root containing this skill (`../../..` from this skill directory). Read its `AGENTS.md` and `docs/repositories.md`; use `docs/agent-guide.md` if workspace or instruction discovery is unclear. Do not assume hub instructions auto-load inside a sibling Git root.

Choose the owner before editing: web for serving/player/coaching UI and its role workers; processing for the video executor/detector/installed worker; server for scheduling/leases/publication; lab for benchmark/model/signal specifications. Keep cross-repo decisions here and actual code in the component.

Read `docs/development.md` for the relevant runtime, then the current component instructions/contracts. Preserve existing local changes. For identity/feedback work, read `docs/domain-model.md`; for proposed research changes, check `docs/decisions.md` so plans are not mistaken for implemented APIs.

Use isolated state and synthetic media for tests. Keep original-time anchors, output-specific judgments, metadata and history stable unless an explicit compatible change is required. Run the affected owner tests; use mobile/desktop checks for UI changes and producer/consumer checks for contract changes.

Review the entire patch against `docs/privacy.md` before external publication. Report actual tests and changed refs. Distinguish implemented, published and deployed. The user's scope controls publication/deployment; this skill does not expand it. Use the separate release skill/runbook only for requested releases.
