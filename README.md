# Cycode scan template for Azure Pipelines

A centralized Cycode scanning pipeline template plus a minimal consumer pattern. Each app team adds a small consumer YAML (~25 lines, parameters only) to their existing Azure Pipelines setup; the scan logic, reporting, and gating all live in one central template that evolves independently.

## What this delivers

- **Delta scans on push.** Each push triggers a Cycode scan of only the new commits since the previous completed build of the same pipeline on the same branch (any prior completed run anchors the range — pass or fail). Falls back to a full scan if no prior build exists.
- **Multiple scan types in one pipeline run.** Any subset of `secret`, `sast`, `sca`, `iac`. Each scan produces an HTML report, a CSV export, a raw JSON, and a Markdown summary on the build's Summary tab.
- **Configurable gate.** Build fails when findings exceed the chosen severity threshold, or runs in report-only mode if preferred.
- **Runs on Microsoft-hosted `windows-2022`.**

## Repo layout

```
templates/cycode-scan.yml             Centralized template (the only thing users reference)
scripts/cycode-summary.py             Generates Markdown / HTML / CSV from Cycode JSON output
azure-pipelines-template-consumer.yml Sample consumer YAML for app teams to copy
```

## Template parameters

| Parameter | Default | Description |
|---|---|---|
| `scanTypes` | `[secret]` | Any subset of `secret`, `sast`, `sca`, `iac`. Scans run in the listed order. |
| `severityThreshold` | `"high"` | `info` \| `low` \| `medium` \| `high` \| `critical`. Findings at or above this severity count toward the gate. |
| `blockOnFindings` | `true` | `true` fails the build on findings above the threshold; `false` scans and publishes reports without failing. |
| `vmImage` | `"windows-2022"` | Microsoft-hosted image used by the scan job. Validated on `windows-2022` and `windows-latest`. |
| `scanMode` | `""` (auto) | Leave empty for auto-detect: CI/PR triggers run a diff scan; manual/scheduled runs scan the full repo. Override with `"full"` or `"diff"` to force. |

## Scan behavior at a glance

| Trigger | Scan mode | What gets scanned |
|---|---|---|
| Push (`IndividualCI`) | diff | Commits since the previous completed build of this pipeline on this branch (pass OR fail — any prior completed run anchors the range) |
| Pull request | diff | Commits in the PR |
| Manual run | full | The entire repository working tree |
| Scheduled run | full | The entire repository working tree |
| First-ever run | full (fallback) | No prior build exists yet, so diff has no baseline |

## What gets published per scan run

For each `scanType` enabled, the pipeline produces:

- A **Build Summary** tab with a Markdown report (severity counts, top findings, links to the Cycode Console).
- A pipeline **artifact** named `cycode-<scanType>-report` containing:
  - `cycode-<scanType>-report.html` — interactive filterable report
  - `cycode-<scanType>-report.csv` — Excel-friendly export
  - `cycode-<scanType>.json` — raw Cycode CLI output
  - `Cycode <Type> Summary.md` — copy of the build summary

## Setup (one-time, per Azure DevOps project)

### 1. Add Cycode credentials as an ADO Library variable group

In the target ADO project: **Pipelines → Library → + Variable group**.

- Name: `cycode-credentials`
- Add two variables, both marked **secret**:
  - `CYCODE_CLIENT_ID`
  - `CYCODE_CLIENT_SECRET`

### 2. Host the template and script in a central repo

Copy `templates/cycode-scan.yml` and `scripts/cycode-summary.py` into your central pipeline-templates repo (the same one you already use to host shared build templates).

Recommended path inside that central repo:
```
templates/cycode-scan.yml
scripts/cycode-summary.py
```

### 3. Add a consumer YAML to each target repo

In the app team's repo, add a pipeline YAML referencing the template. Use `azure-pipelines-template-consumer.yml` in this repo as the example. The critical pieces:

```yaml
trigger:
  branches:
    include: [main]              # adjust to whichever branches should trigger CI

name: $(Build.SourceBranchName)_$(Date:yyyyMMdd)$(Rev:.r)

variables:
  - group: cycode-credentials

resources:
  repositories:
    - repository: security-templates                          # this alias name is REQUIRED
      type: git
      name: <YourProject>/<YourTemplatesRepo>                 # the repo from step 2
      ref: refs/heads/main                                    # or refs/tags/v1 to pin

extends:
  template: templates/cycode-scan.yml@security-templates
  parameters:
    scanTypes: [secret, sca, iac, sast]   # any subset of: secret, sast, sca, iac
    severityThreshold: "high"             # info | low | medium | high | critical
    blockOnFindings: true                 # true = fail build on findings; false = scan and report only
    # vmImage: "windows-2022"             # OPTIONAL — validated on windows-2022 and windows-latest
    # scanMode: ""                        # OPTIONAL. Leave unset for diff scan on push or full scan on manual run. Force mode with "full" | "diff"
```

Register the file as a pipeline in Azure DevOps (Pipelines → New pipeline → Azure Repos Git → select the repo → Existing Azure Pipelines YAML).

### 4. Cross-project setup (only if the templates repo lives in a different ADO project)

Skip this section if the templates repo and the consumer repo are in the **same Azure DevOps project**. Within one project, the project's Build Service identity has implicit Read on every repo and no extra configuration is required.

For one template serving consumers across projects in the same org, three one-time setup steps may be required (depending on your environment):

**a. Grant the consumer's Build Service Read on the templates repo.** In the project that hosts the templates repo:

1. **Project Settings → Repos → Repositories**.
2. Click the templates repo (the one containing `templates/cycode-scan.yml`).
3. Top tabs → **Security**.
4. Click **+ Add** under *Azure DevOps Groups*. Search for and add **`Project Collection Build Service Accounts`** — this is the org-wide group containing every project's Build Service identity, so one grant covers every present and future consumer.
5. With that entry selected, set **Read** = **Allow**. Save.

   Alternatively, add the specific `<ConsumerProject> Build Service (<your-org>)` identity instead of the collection group. Narrower, more grants required as you add consumer projects.

**b. Disable "Limit job authorization scope" at the org level.** Microsoft enabled this policy by default in newer ADO orgs; it blocks per-project Build Service identities from reaching cross-project resources at the auth layer, *before* repo permissions are even checked.

1. **Organization Settings → Pipelines → Settings** (URL: `https://dev.azure.com/<your-org>/_settings/pipelinessettings`).
2. Disable both:
   - **Limit job authorization scope to current project for non-release pipelines** → Off
   - **Protect access to repositories in YAML pipelines** → Off (or leave on and pre-declare every cross-project repo as a permitted resource — disabling is simpler).
3. Save.

**c. Confirm the same settings at the consumer project level.** Org-level settings can be locked or unlocked for project-level override; either way, also check the consumer project:

1. In the consumer project: **Project Settings → Pipelines → Settings**.
2. Verify the same two toggles are **Off**. Save if you changed anything.

**Failure mode if steps b/c are skipped:** the checkout step fails with `TF401019: The Git repository with name or identifier <repo> does not exist or you do not have permissions for the operation you are attempting.` This is the symptom that points back to the auth-scope policy — repo Read permission alone is not enough when the policy is on.

### First-run prompt

The first time the consumer pipeline runs, Azure DevOps pauses with:

> "This pipeline needs permission to access a resource before this run can continue"

Click **View → Permit** for the `security-templates` repository resource and the `cycode-credentials` variable group. This authorization is one-time per pipeline.

## Notes

- The `security-templates` repository alias name is hardcoded in the template's `- checkout: security-templates` step. Customers should use this exact alias name in the consumer's `resources.repositories` block.
- The template checks out the security-templates repo at runtime using a separate physical directory (`s/_security-templates`) so that same-repo scenarios work alongside cross-repo production usage.
- The diff scan resolves its BASE commit via the Azure DevOps Build REST API, using `$(System.AccessToken)`. The template adds `persistCredentials: true` on the self checkout to enable this — no customer configuration required.
