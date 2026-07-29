# code-review-automation-test

A harness for evaluating free and low-cost **code review automation tools**
— SAST, SCA, AI PR review and DAST — against real, mostly AI-generated web
applications.

This repository contains **workflows and documentation only**. It hosts no
application source of its own and no third-party source. The application under
test is cloned from its own upstream repository at workflow runtime, scanned
inside the ephemeral GitHub Actions runner, and discarded when the run ends.
Results are published as **run artifacts**.

The first target is a public third-party Next.js 15 / React 19 / TypeScript
application. Its upstream repository publishes no licence file, so no copy of it
is redistributed here — see
[`docs/adr/ADR-001`](docs/adr/ADR-001-repository-purpose-and-target-model.md)
for the full rationale and its consequences for the tools under test.

---

## Workflows

Both are **manual** (`workflow_dispatch`). Run them from the **Actions** tab →
pick the workflow → **Run workflow**. There is no push or pull_request trigger:
with no application source committed here, an automatic scan would have nothing
meaningful to analyse.

### `Code Audit (static + SCA)` — `.github/workflows/audit-static.yml`

Clones the target and runs three scanners over its source tree:

| Job | Tool | Covers |
|---|---|---|
| `SAST - Semgrep` | Semgrep OSS | injection, secrets, OWASP Top Ten, JS/TS/React/Node rule packs |
| `SCA - OSV-Scanner` | OSV-Scanner | known vulnerabilities in the lockfile |
| `SCA + secrets - Trivy` | Trivy | dependency vulns, secrets, misconfiguration |

A final `Findings summary` job downloads every SARIF artifact and writes a
per-file result count into the run summary.

**Inputs**

| Input | Default | Meaning |
|---|---|---|
| `target_repo` | `ILoveBacteria/english-vocab-memorization` | `owner/repo` to clone and scan. Clear it to scan **this** repository instead, which additionally uploads SARIF to Security → Code scanning. |
| `target_ref` | `ed3b2bdfa8918e317a46b15766d5d0a3d30e88c9` | Branch, tag or SHA of `target_repo`. Pinned to a commit by default so repeat runs are reproducible and comparable. Ignored when `target_repo` is empty. |

**Artifacts:** `sarif-semgrep`, `sarif-osv-scanner`, `sarif-trivy`, and
`sarif-all` (the combined bundle).

Snyk Code is present but commented out; enable it by adding a `SNYK_TOKEN`
repository secret and uncommenting the job.

### `Code Audit (DAST)` — `.github/workflows/audit-dast.yml`

Clones the target, installs and builds it, boots it inside the runner, waits for
it to respond, then attacks that in-runner instance with OWASP ZAP (baseline and
full active scan), Nuclei and Nikto.

**Inputs**

| Input | Default | Meaning |
|---|---|---|
| `target_repo` | `ILoveBacteria/english-vocab-memorization` | `owner/repo` of the application to clone, build and scan. Required. |
| `target_ref` | `ed3b2bdfa8918e317a46b15766d5d0a3d30e88c9` | Branch, tag or SHA of `target_repo`. |
| `run_zap_baseline` | `true` | Run the ZAP passive baseline scan. |
| `run_zap_full` | `true` | Run the ZAP full **active** scan (slow; sends attack payloads). |
| `run_nuclei` | `true` | Run Nuclei. |
| `run_nikto` | `true` | Run Nikto. |
| `zap_spider_minutes` | `5` | Minutes ZAP may spend spidering, per scan. |

**Artifacts:** `dast-reports` — ZAP HTML/JSON/XML/Markdown, Nuclei
JSONL/SARIF/text, Nikto HTML/JSON, plus the application's own log.

Each tool runs with `continue-on-error`, so one failing scanner does not hide
the others' results, and the job has a 120-minute timeout.

---

## Safety model

The DAST workflow sends real attack traffic, so its blast radius is fixed by
construction rather than by convention:

- **The scanned URL is not an input.** It is hardcoded to the instance booted
  inside the runner (`http://127.0.0.1:3000`). `target_repo` selects which
  *source* gets built; it can never redirect attack traffic at an external host.
- **Manual trigger only.** Active scans are never triggered incidentally by a
  push or a pull request.
- **The real backend is never touched.** The target app expects a hosted
  Supabase project — a third party. It is booted with dummy, non-routable
  Supabase environment values instead, so pages still render while data and auth
  calls fail harmlessly.
- **No outbound callbacks.** Nuclei runs with `-ni`, disabling interactsh, so no
  OAST traffic leaves the runner.
- **Everything is ephemeral.** The cloned source and the running app live only
  for the duration of the run.

---

## Repository layout

```
.
├── .github/workflows/
│   ├── audit-static.yml   # SAST + SCA over a cloned target
│   └── audit-dast.yml     # boot the cloned target in-runner, then ZAP/Nuclei/Nikto
├── docs/adr/              # architecture decision records
└── README.md
```

No secrets are committed. Optional tokens (for example `SNYK_TOKEN`) belong in
GitHub Actions repository secrets.
