---
name: courtside-release
description: Publish, deploy and verify an authorized Courtside website, processing-server or worker release using the component runbooks. Use for Courtside release/upgrade/recovery work, not ordinary implementation or unrelated services.
---

# Courtside release

Locate the hub root (`../../..` from this skill directory). Read `AGENTS.md` and `docs/deployment.md`, then the affected component's current release workflow and installation instructions. Read `docs/operations.md` for worker/job recovery and `docs/privacy.md` before publishing evidence.

Apply existing user authorization without inventing another approval gate. Implement-only work does not authorize deployment; an explicit routine deployment request does. A release does not authorize new exposure, volume deletion, access expansion or unrelated worker changes.

Identify the exact source/tree, CI result, release trigger and existing private deployment target. Website and server triggers can differ. Require completed release checks and verify the published image digest, platform and source revision before switching an image. Do not claim a tag/release page proves image publication.

Preserve active jobs and all persistent state. Back up the complete stopped website state and pinned evidence; a single website database copy is incomplete. Never open live SQLite over SMB. Preserve names, mounts and existing access boundaries, change the image through the established deployment route, and avoid Clean/prune/volume removal.

Use GitHub APIs/connectors and normal Git tooling where sufficient. Narrow UI fallbacks are acceptable for missing capabilities. Never extract browser sessions; wait for manual unlock when the computer-use tool requires it.

Verify health, actual runtime behavior, retained metadata/history and relevant workers. Use read-only production checks and isolated fixtures for mutation tests. Keep the deployment record private. If recovery could discard newer writes, surface that decision rather than automatically restoring an old snapshot. Report any incomplete step precisely.
