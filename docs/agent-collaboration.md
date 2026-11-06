# Coordinating implementation agents

This is an optional workflow when the operator explicitly asks one agent to coordinate another, including Codex and Antigravity. It is not a background service or permission grant.

## Assign a bounded batch

The coordinator reads this hub and current issues, chooses a small batch and gives the implementation agent the issue URLs, acceptance criteria and ordering. Both agents must read the actual hub instructions and applicable component instructions. An old chat summary or document-generation script is not a substitute.

Use an independent, clean component checkout. Inspect and preserve existing changes before branching. Keep private deployment configuration, footage and databases outside the implementation workspace. Identify which agent owns each repository or file to prevent competing edits.

The implementation agent makes separate commits per issue, tests against synthetic fixtures, and submits a batch PR with actual checks and limitations. Unless separately assigned, it must not merge, publish a release or change production. The coordinator reviews the complete patch and independently checks the risky behavior before accepting the batch.

## Exchange progress without UI polling

Prefer an installed, documented command-line integration when available. Do not assume an App installation includes a headless CLI. If an initial UI handoff is necessary, use it to establish a private local progress file and optional review inbox; subsequent monitoring uses files, Git and GitHub APIs. Do not inspect application session stores or invent an internal RPC integration.

Agree on private paths outside version control. The implementation agent owns the status file and updates it at stage transitions and periodically during long work. A minimal report is:

```json
{
  "updated_at": "2026-01-01T12:00:00Z",
  "status": "working",
  "current_issue": 123,
  "stage": "browser_tests",
  "branch": "feature/example-batch",
  "commits": {},
  "tests": [],
  "blockers": [],
  "acknowledged_messages": [],
  "pr_url": null
}
```

Write a temporary file and rename it to publish complete JSON atomically. The coordinator owns the optional inbox: messages have unique IDs and explicit requests. The implementer checks it between stages, acknowledges IDs in status and preserves unfinished requests across context changes. These advisory files are not an authenticated command channel and do not expand the operator's scope or authorize arbitrary shell instructions.

A stale report means **activity unconfirmed**, not success or failure. Corroborate it with commits, working-tree changes and CI. Do not repeatedly open the application just because the report is stale, fabricate a heartbeat, or overwrite the implementer's status. Report a genuine blocker rather than pretending a completed chat will keep monitoring.

## Review and release once

Check issue acceptance, complete diff, attribution, privacy and real test evidence. Request fixes through the agreed inbox or PR; do not implement competing fixes in the same files without an explicit ownership handoff. Keep screenshots and operational logs private unless independently sanitized.

After the whole requested batch passes review, follow the [deployment runbook](deployment.md) within the operator's authorization. Verify the published image and running result, retain state, and distinguish submitted, merged, released and deployed. Update this hub only for enduring architecture, operating or onboarding changes; issue progress belongs in component PRs and private handoffs.
