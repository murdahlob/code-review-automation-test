# ADR-001 — Repository purpose and hybrid target model

**Date:** 2026-07-29  
**Status:** Accepted

---

## Context

This repository exists to evaluate free / low-cost code review automation tools
(SAST, AI PR review, SCA, DAST) against mostly AI-generated web applications.
Several of the tools under test are GitHub-native and only operate on the
repository they are installed on: Dependabot scans this repo's manifests,
CodeRabbit/DeepSource review this repo's pull requests, SonarQube Cloud imports
this repo as a project, and SARIF uploaded to the Security tab only maps
correctly to files that exist in this repo's tree. Their free tiers — and free
GitHub code scanning and unlimited Actions minutes — require the repo to be
public.

---

## Decisions

### 1. The app under test is committed as this repository's own content

Rather than cloning a target at scan time only, the target application's source
is committed at the repo root. This is what makes the GitHub-native tools work
correctly and keeps the Security → Code scanning tab an accurate unified view
of all scanner findings.

### 2. Scan workflows are additionally parameterized by `target_repo`

The audit workflows accept a `workflow_dispatch` input `target_repo` defaulting
to this repo's own tree, so other targets can be scanned ad hoc without
re-populating the repo. For such targets only the run artifacts are meaningful
— the Security tab, Dependabot, and the PR reviewers still reflect only this
repo's own content.

### 3. Static and dynamic scanning are split into separate workflows

`audit-static.yml` (SAST + SCA) is safe and cheap, running on push/PR.
`audit-dast.yml` boots the app inside the ephemeral runner and attacks it
(ZAP/Nuclei/Nikto) — destructive to its target instance, long-running, and
therefore `workflow_dispatch` only. Attack traffic never leaves the runner.

---

## Consequences

- The repo's git history is the trial's record of scanner configuration; the
  target app's own upstream history is not preserved here (the upstream repo
  remains the source for that).
- Replacing the app under test means replacing the repo content, or adding a
  second target via the `target_repo` parameter with artifact-only results.
- Findings published in the Security tab describe intentionally vulnerable-ish,
  mostly AI-generated code under evaluation — not code intended for production
  use.
