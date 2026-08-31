---
id: adr-0004
title: ADR-0004 — Repoint the direct parent and repository identity to the current account
status: proposed
updated: 2026-09-01
---

# ADR-0004: Repoint the direct parent and repository identity to the current account

| Field | Value |
|-------|-------|
| Status | proposed |
| Date | 2026-09-01 |
| Deciders | repository owner |
| Author | Claude Code (AI agent) |
| Supersedes / Superseded by | Supersedes the direct-parent clause of ADR-0003 |

## Context

This repository has moved to the `ea-Mitsuoka` GitHub account. `git remote get-url
origin` is `https://github.com/ea-Mitsuoka/terraform-gcp-template.git`. Its inheritance
metadata still names the former `Yukihide-Mitsuoka` account, both as its direct parent
and as its own identity.

`ea-Mitsuoka/ai-dev-foundation` and `Yukihide-Mitsuoka/ai-dev-foundation` are distinct
repositories under distinct accounts, so the recorded parent is not reachable from the
maintained foundation through GitHub rename redirection. Three effects follow:

- Scheduled Template Sync reads `Yukihide-Mitsuoka/ai-dev-foundation`, a repository the
  owner no longer updates. A stale parent produces no error — only an absence of
  incoming changes.
- `make doctor` reported `Root README ownership is invalid (ADR-0011)`, because
  `scripts/readme_ownership.py` compares the marker against `remote.origin.url`.
- `.github/CODEOWNERS` routes every review to `@Yukihide-Mitsuoka`, who no longer owns
  this repository, so required-reviewer routing cannot resolve.

ADR-0003 states that this repository MUST name `Yukihide-Mitsuoka/ai-dev-foundation` as
its only direct parent. Accepted ADRs are never edited, so that clause needs a
superseding decision.

This repository is also a parent. It exports a Terraform family contract to
`secure-ga4-bq-template` from the owner-qualified root
`.ai/contracts/templates/yukihide-mitsuoka/terraform-gcp-template/`. ADR-0014 in the
foundation derives that path from the exporting repository's owner, and
`scripts/template_inheritance.py` enforces the derivation: a `template` agent-profile
input must live under `.ai/contracts/templates/<owner>/<repository>/`, lowercased. The
directory name is therefore part of the contract, not a naming preference.

## Options considered

### Option 1: Do nothing

Leave both the parent reference and this repository's own identity on the former
account. Nothing breaks immediately, and the recorded parent still exists.

It leaves `make doctor` failing, leaves review routing pointed at a non-owner, and lets
this repository drift from the maintained foundation indefinitely.

### Option 2: Repoint the direct parent only

Change `.github/inheritance/manifest.json`, `lock.json`, `agent-profile.json`, and the
Template Sync source, and leave this repository's own owner-qualified identity on the
former account.

This restores synchronization with one small diff. It leaves the README ownership check
failing, leaves CODEOWNERS broken, and leaves the exported contract root claiming an
owner that no longer holds this repository — so a new grandchild bootstrapped from the
export would inherit the wrong owner-qualified path.

### Option 3: Repoint the parent and this repository's identity together

Additionally rename the exported contract root to
`.ai/contracts/templates/ea-mitsuoka/terraform-gcp-template/`, update the manifest
ownership entry and `.templatesyncignore` that must cover it, and update CODEOWNERS, the
README ownership marker, the repository-facts overlay, the PR-size policy constants, and
the project-owned tests that pin these values.

This is a larger diff and requires a coordinated follow-up in `secure-ga4-bq-template`,
which declares the old path in its own `inherited_paths`. It leaves no half-migrated
state behind.

## Decision

Adopt Option 3.

This repository MUST name `ea-Mitsuoka/ai-dev-foundation` as its only direct parent,
superseding the corresponding clause of ADR-0003. Every other constraint of ADR-0003
stays in force: one first-parent commit per reviewed PR, protected child-owned paths,
and the Terraform family profile reaching `secure-ga4-bq-template` only through this
repository.

The accepted lock commit is unchanged.
`53fadbe8d8dc5dd97a7dfb11d4ab17b2ba308d65` exists in
`ea-Mitsuoka/ai-dev-foundation`, so the repoint advances no inheritance state and
re-verifies no blob.

This repository MUST publish its Terraform family contract from
`.ai/contracts/templates/ea-mitsuoka/terraform-gcp-template/`. `secure-ga4-bq-template`
MUST adopt the renamed root in its own reviewed PR after this one merges; until then it
keeps the old path and its inherited copy is stale but valid.

An owner-qualified reference that records history MUST NOT be rewritten. `CHANGELOG.md`,
existing `.ai/decision-log.md` rows, the accepted ADR-0003 body, and the issue, PR, and
Actions links in `docs/handoff.md` keep the account that hosted them.

Module and workflow sources are outside this decision.
`Yukihide-Mitsuoka/terraform-gcp-modules` in `infra/` and `Makefile`, and
`Yukihide-Mitsuoka/gcp-cicd-workflows` in `CLAUDE.md`, are versioned artifact references
pinned by tag rather than inheritance edges. Repointing them requires first verifying
that each pinned tag exists under the new account, and a wrong move breaks
`terraform init`.

## Consequences

**Positive:**

- Scheduled Template Sync reads the foundation the owner maintains.
- `make doctor` passes: `readme ownership: OK: ea-Mitsuoka/terraform-gcp-template`.
- Review routing resolves again, because CODEOWNERS names the account that owns the
  repository.
- A grandchild bootstrapped from the exported contract receives the correct
  owner-qualified path, so the migration does not reproduce itself downstream.

**Negative:**

- `secure-ga4-bq-template` must land a coordinated PR. Between the two merges its
  `inherited_paths` names a directory this repository no longer publishes, so a Template
  Sync run in that window carries no family-contract update.
- The repository now contains both spellings, and a reader must distinguish a current
  reference from a historical link.
- `docs/handoff.md` keeps unrelated staleness — its recorded lock, baseline, and parent
  target predate the current `main`. This decision corrects only the owner fields.
- Rollback is a reviewed revert of this PR plus the matching revert in
  `secure-ga4-bq-template`, in child-first order.

**Follow-ups:**

- Migrate `secure-ga4-bq-template` to the renamed contract root and to
  `ea-Mitsuoka/terraform-gcp-template` as its direct parent.
- Verify the `terraform-gcp-modules` and `gcp-cicd-workflows` tags under the new account,
  then repoint those sources in a separate PR.
- Refresh the stale restart point in `docs/handoff.md`.
