# ADR-001 — Repository purpose and target model

**Date:** 2026-07-29  
**Status:** Accepted

---

## Context

This repository exists to evaluate free / low-cost code review automation tools
(SAST, AI PR review, SCA, DAST) against mostly AI-generated web applications.

The evaluation needs a real target application. The first target is a public
third-party Next.js application on GitHub. Its upstream repository publishes no
licence file, so no licence grants permission to redistribute its source. This
repository therefore does not host a copy of it.

That constraint interacts directly with the tools under test. Several of them
are GitHub-native and only operate on the repository they are installed on:
Dependabot scans the manifests committed to a repo, CodeRabbit and DeepSource
review a repo's own pull requests, SonarQube Cloud imports a repo as a project,
and SARIF uploaded to Security → Code scanning only maps correctly to files
that exist in that repo's tree. The remaining tools are ordinary CLIs that will
scan whatever source tree they are pointed at, regardless of where it came
from.

---

## Decisions

### 1. The target application is cloned at workflow runtime, never committed here

The audit workflows check the target out from its upstream repository at run
time (`actions/checkout` with a `repository` input) and scan that working copy
inside the ephemeral runner. Nothing of the target's source is committed to
this repository, and the runner's copy is discarded when the run ends.

The reason is licensing: upstream publishes no licence file, so this repository
redistributes no copy of that code. The alternative — vendoring the app at the
repo root — would have made the GitHub-native tools work, but it would mean
republishing third-party source without a licence permitting it. Tool coverage
does not justify that, so the runtime-clone model is used for phase 1.

### 2. The target is selected by workflow inputs and pinned by default

Both workflows take `target_repo` and `target_ref` `workflow_dispatch` inputs.
`target_ref` defaults to a specific upstream commit SHA rather than a branch
name, so repeat runs scan identical bytes and results stay comparable across
tools and across time. Either input can be overridden per run to scan a
different target or a newer commit.

### 3. Results are published as run artifacts only

Because the scanned paths do not exist in this repository, SARIF is uploaded as
run artifacts and not to the Security tab, where it would be dropped or
mis-mapped. A summary job counts findings per SARIF file into the run summary.

### 4. The self-scan path is retained but dormant

The static workflow keeps its original mode where an empty `target_repo` scans
this repository's own tree and uploads SARIF to Security → Code scanning. It is
dead in phase 1 — this repo holds only workflows and Markdown — but it is the
path a future target would use if that target's licence permits hosting its
code here, so it is preserved rather than deleted.

### 5. No push or pull_request trigger on the audit workflows

With no application source in this repository, a push-triggered scan would
analyse Markdown and YAML and report nothing of value. `workflow_dispatch` is
the trigger for both workflows.

### 6. Static and dynamic scanning stay in separate workflows

`audit-static.yml` (SAST + SCA) is cheap and safe. `audit-dast.yml` clones and
builds the target, boots it inside the ephemeral runner and attacks it
(ZAP / Nuclei / Nikto) — destructive to its target instance and long-running.
Its scanned URL is hardcoded to the in-runner instance and is deliberately not
an input, so the workflow cannot be aimed at production, shared, or
third-party infrastructure. `target_repo` selects only which source is built.

---

## Consequences

- **Five tools are unavailable in phase 1**, because each requires the code to
  live in this repository: **Dependabot** (scans committed manifests),
  **CodeRabbit** and **DeepSource** (review this repo's own pull requests),
  **SonarQube Cloud** (imports a repo as a project), and the **Security → Code
  scanning tab** as a unified cross-tool view. Substitutes are used where they
  exist — self-hosted SonarQube for SonarQube Cloud, OSV-Scanner and Trivy for
  Dependabot's SCA coverage — and the gap is recorded as a finding of the trial
  rather than worked around.
- **The CLI-shaped tools are unaffected**: Semgrep, Snyk Code, OSV-Scanner,
  Trivy, ZAP, Nuclei and Nikto all scan a path or a URL and do not care that the
  source arrived by clone rather than by commit.
- **The repository stays public.** It hosts no third-party code, so being public
  raises no licensing question, and public repositories get unlimited GitHub
  Actions minutes — which the long DAST runs depend on.
- **Comparability rests on the pinned SHA.** Runs that override `target_ref`
  are not directly comparable with the pinned baseline, and results should
  record the ref they were produced from.
- **Every run pays clone and build cost.** The DAST workflow installs
  dependencies and builds the app on each run, which is slower than scanning a
  committed tree and depends on upstream remaining reachable.
- **Findings describe third-party, mostly AI-generated code under evaluation.**
  They are observations about a sample application chosen for this trial, not a
  security advisory about anyone's production system.
