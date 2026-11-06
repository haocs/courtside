# Contributing

## Choose the owner

Use [the repository map](docs/repositories.md). Put application code, component tests and exact API/schema changes in the owning repository. Put shared architecture, operational decisions and onboarding here. A cross-component issue can live in this hub and link focused component issues/PRs; component bugs should stay with their implementation.

Check current default branches and existing local work before editing. Keep repositories independently versioned. Do not vendor their source, add submodules, or copy runtime configuration into this documentation hub.

## Change workflow

1. State the user-visible behavior and affected contracts. Distinguish current behavior, the proposed change and deferred work.
2. Read the current component README, scoped instructions and source. Use [the reviewed source snapshot](docs/repositories.md#reviewed-source-snapshot) for historical context, not as an instruction to downgrade newer code.
3. Implement and test in the component. For contract changes, test producer and consumer compatibility, stale versions, retries and reprocessing identities where relevant.
4. Update shared docs when ownership, setup, storage or operations change. Link component contracts instead of duplicating whole schemas.
5. Review the complete diff and any new files using [privacy checks](docs/privacy.md). Documentation examples must be generic and independently authored.
6. Open the appropriate PR and identify actual validation. Release/deploy only within the user's requested scope.

Documentation changes in this repo need link, consistency and privacy review; there is no application test suite to run here. Validate each bundled skill's frontmatter and referenced runbooks when editing it. Do not add an executable product or CI implementation simply to make this docs repo look like a code repo.

## Cross-repository coordination

Keep issue/PR references in a cross-repository tracking issue when a feature needs several components. State dependency order and rollout/rollback compatibility. Do not use a merged website feature as proof that a planned research service or training executor exists.

Summarize decisions in [decisions](docs/decisions.md); keep speculative work in [roadmap](docs/roadmap.md). Refresh the reviewed-source snapshot when re-auditing the components. Retain public source links for the evidence behind a changed instruction.

## Review and handoff

A handoff should state changed repositories and refs, tests actually run, remaining work, and whether anything was published or deployed. Installation-specific image records, host addresses, storage mappings and backups belong in the operator's private operations record, not a public issue or Markdown example.

Public visibility is not a license grant. Respect each component's license/dependency notices; this hub does not relicense upstream code, model artifacts or media.
