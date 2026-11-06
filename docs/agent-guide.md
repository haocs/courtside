# New-agent onboarding

## First request to an agent

Use this task-independent starting prompt, then add the actual feature or issue:

> Start with courtside/AGENTS.md and docs/repositories.md. Read the current component instructions and contracts for this task. Inspect local/remote state before editing. Work in the owning code repository; keep this hub documentation-only. Use synthetic fixtures and external private config. Preserve original-time identities and feedback history. Implement and verify within the requested scope, and report whether anything was published or deployed.

## Workspace and access

Clone the [sibling layout](development.md#workspace), open this hub as the orientation entry point and attach/open the needed component repositories through the agent/IDE's normal workspace controls. Make the required component writable explicitly if the environment initially grants access only to this hub. Do not change global permissions, nest source in NAS media folders, or rely on an old chat's runtime/interpreter.

Each component is its own Git root. After moving to a sibling repository, hub rules/skills are not guaranteed to be auto-discovered. Read this hub's AGENTS and relevant skill/runbook explicitly, then load that component's local scoped instructions. Do not copy a large, diverging policy file into every repo just to simulate inheritance.

The starting environment needs Git and the toolchain for the affected component. Routine code changes need neither a live NAS nor an external AI account. A release needs the actual authorized repository/deployment tooling and the operator's private installation record; this public repo deliberately cannot supply those values.

## Codex

The root [AGENTS.md](../AGENTS.md) is the instruction entry point. Repository skills are in `.agents/skills/`. From this hub, Codex can discover them; start a fresh session/reload skill discovery if a new skill is not listed. When working across Git roots, explicitly reference the skill file if it is not available in the current selector. Do not assume an agent has loaded it merely because it exists in a sibling directory.

Use `$courtside-change` for component work and `$courtside-release` for an authorized release/deployment. The skills require no proprietary bridge, application runtime, provider API or automatically installed MCP dependency.

These locations follow [OpenAI's instruction guidance](https://learn.chatgpt.com/docs/agent-configuration/agents-md) and [skill guidance](https://learn.chatgpt.com/docs/build-skills). Re-check product documentation if discovery behavior differs in a future version; the Markdown runbooks remain usable manually.

## Antigravity

Current official documentation supports workspace `AGENTS.md`/`GEMINI.md` and workspace `.agents/skills/`. This repo supplies the same AGENTS instructions plus a tiny [GEMINI.md](../GEMINI.md) pointer; there is no separate policy copy.

Open the hub as a project, verify active instructions/skills in the product's Customizations view and then attach the component workspace required for editing. Use the discovered skills or explicitly read their SKILL.md files. A product variant/version that does not auto-load one of these files can still follow AGENTS and the runbooks through the starting prompt; verify discovery rather than claiming cross-agent execution was tested.

See official [rules](https://antigravity.google/docs/rules/) and [skills](https://antigravity.google/docs/skills/) documentation for current App/CLI/IDE discovery behavior. No global machine configuration or extension installation is required by this hub.

## How to choose tools

Use repository APIs/connectors for issues/PRs/files when available; normal Git tooling is appropriate for local checkout/worktree state. Use product UI only for genuinely missing capabilities, such as a particular release/project control in an available environment. Do not turn UI-only access into a session-data extraction mechanism. Existing authorization still applies; a skill is not additional permission.

Use component CLIs for worker status/restarts, and the existing deployment route for containers. Keep training and inference/provider setup separate from an ordinary website task. Do not add parallel agents or competing services simply because tooling supports them.

## A self-contained handoff

For an explicitly requested coordinator/implementation-agent workflow, follow [agent collaboration](agent-collaboration.md), including file-based progress and review before a batch deployment.

Record the following in a private task handoff when necessary:

- User goal, accepted decisions and outstanding questions.
- Changed component refs and whether the working trees contain unrelated work.
- Tests actually run, observed failures and platform limits.
- Publication/deployment state and the next concrete step.
- A reference to the operator's private deployment record when needed, without copying its contents here.

Public handoffs may contain public issue/PR/source links and sanitized implementation evidence. They must not contain personal data, access/session data, live paths, footage or raw logs. Never describe an interrupted deployment as complete.
