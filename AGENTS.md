# Courtside agent entry point

Read this file when starting work on Courtside, including when another component is your working directory. This hub is documentation-only; implement application changes in the owning component.

## Orientation

1. Read [repository ownership](docs/repositories.md), [architecture](docs/architecture.md) and [current decisions](docs/decisions.md). Read the domain model for changes involving identity, metadata, Moments, feedback or reprocessing.
2. Follow [development setup](docs/development.md) for the target component. Read its README, scoped AGENTS files, exact contracts and current workflow. Inspect its branch, uncommitted changes and remote state before editing.
3. Prefer current implementation and explicit user decisions over stale proposal wording. Report a genuine contract conflict before making an incompatible migration.
4. Treat sibling repositories as independent Git roots. Hub instructions and skills may not load automatically after changing roots: read them explicitly, then respect component-local instructions. See [agent onboarding](docs/agent-guide.md).

## Project invariants

- Keep originals private and immutable. The website reads only intentionally published media; its mutable state is separate.
- Deploy website and processing-server code as versioned container images. NAS media directories are not development checkouts.
- Install workers independently of any editor, chat workspace or agent runtime. Preserve the video-processing worker and its queue when changing coaching workers.
- Processing APIs carry metadata and artifact references. Large media moves through storage. Server leases and publication decisions are authoritative.
- Anchor Moments and time-based labels to original-recording time. Clip usefulness and cut judgments belong to specific outputs; do not carry them to new cuts by approximate overlap.
- Keep serving metadata, processing evidence and accepted benchmark labels distinct. AI advice or a downvote is not automatically training truth.
- Preserve compatible URLs, original metadata, preferences, histories, evidence and existing worker/job state when updating components.
- Never open a live NAS SQLite database through an SMB client mount. Use local/native database operations or verified stopped-state copies.

## Work and release behavior

Use GitHub APIs/connectors or established Git tooling for repository work. Use browser/computer controls only for capabilities unavailable through those tools. Do not extract browser sessions or copy access material to work around missing capabilities.

Use the user's existing authorization. A request to implement is not a deployment request; a deployment request authorizes the routine verified update within its stated scope. Do not introduce repeated approval steps for already-authorized work. New access, destructive data removal or unrelated service changes require their own justification and applicable approval.

Run relevant component tests and inspect mobile/desktop behavior for UI changes. Mark unverified results honestly. Never call a release deployed merely because its source or image was published. Follow [deployment](docs/deployment.md) for digest verification, backups and live checks.

Report the component, commit/PR, checks, implementation status and deployment status. Keep private installation values out of reports destined for this repo. Do not overwrite another contributor's changes or silently activate an experimental output.

## Privacy

[Privacy rules](docs/privacy.md) apply to every file and commit. No credentials, access/session data, personal identifiers, real footage, GPS/court data, machine/NAS identities, private paths, runtime databases or raw operational screenshots/logs belong here. Public component URLs and public source revisions are the only project-specific identifiers needed for onboarding.

## Reusable skills

- [.agents/skills/courtside-change/SKILL.md](.agents/skills/courtside-change/SKILL.md): route and validate component changes.
- [.agents/skills/courtside-release/SKILL.md](.agents/skills/courtside-release/SKILL.md): verified releases, deployment and recovery boundaries.

Skills route to the maintained runbooks; they do not grant additional permissions or require a specific agent vendor.
