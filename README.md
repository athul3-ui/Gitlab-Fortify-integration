# fortify-sast-gitlab-ci

A GitLab CI/CD pipeline job that runs a full Fortify SAST scan and uploads results to Fortify SSC automatically handling app and version creation in SSC dynamically, so you never have to set anything up manually in the SSC UI.

---

## What problem this solves

Getting Fortify integrated into a CI/CD pipeline sounds simple until you hit the SSC side of things. The scan itself is straightforward, but uploading results requires a valid SSC version ID, which means you either pre-create every application and version in SSC by hand (tedious at scale), or you automate it. This script does the latter.

Every time the pipeline runs, it checks whether the application and version already exist in SSC. If they do, it reuses them. If they don't, it creates them and uploads. No manual SSC setup, no hardcoded version IDs, no pipeline failures because someone forgot to create a version before merging a new branch.

---

## How it works

The pipeline stage runs nine steps in sequence:

```
Clean → Translate → Scan → Check SSC version exists?
                                ├─ Yes → Upload directly
                                └─ No  → Check app exists?
                                              ├─ Yes → Create version → Commit → Upload
                                              └─ No  → Create app + version → Commit → Upload
```

SSC organises results as **Application → Version → Scan**. The app name is built from your GitLab project name + branch name (`payment-service-main`), and the version is the branch name alone (`main`). This means all branches of the same project share one application entry in SSC, with separate version views per branch — so triage history carries across branches without duplication.

---

## Setup

### 1. Runner requirements

You need a GitLab runner registered on a **Windows machine** with:

- Fortify SCA installed (OpenText/Micro Focus Fortify 26.2.0 or compatible)
- PowerShell available (standard on Windows Server)
- Network access to your SSC server
- The runner tagged as `fortify-slave-1` (or update the `tags:` in the YAML to match yours)

If you're using a self-hosted runner, register it with:
```bash
gitlab-runner register --tag-list "fortify-slave-1" ...
```

### 2. GitLab CI/CD variables

Add these in **Settings → CI/CD → Variables** (mark both as Protected + Masked):

| Variable | Description | Example |
|---|---|---|
| `FORTIFY_SSC_URL` | Full URL to your SSC server | `https://ssc.yourcompany.com/ssc` |
| `FORTIFY_TOKEN` | SSC auth token | Generate in SSC → Administration → Token Management → UnifiedLoginToken |

### 3. Fortify installation path

Check the `FORTIFY_HOME` path in the script matches your installation:

```powershell
$env:FORTIFY_HOME = "C:\Program Files\Fortify\OpenText_SAST_Fortify_26.2.0"
```

Update the version number if you're on a different release.

### 4. How this file fits into your pipeline

`fortify_scan.yml` is a **standalone template file** — it is not your main pipeline file. GitLab does not run it directly. Instead, your main `.gitlab-ci.yml` pulls it in using the `include:` keyword, and then a job in that file `extends:` the base job defined here.

Your repo structure should look like this:

```
your-repo/
├── .gitlab-ci.yml        ← GitLab reads this automatically on every push
├── fortify_scan.yml      ← this file, included by .gitlab-ci.yml
└── src/
    └── ...
```

In your `.gitlab-ci.yml`, add:

```yaml
include:
  - local: 'fortify_scan.yml'   # tells GitLab to merge this file into the pipeline

stages:
  - build
  - test
  - scan
  - deploy

fortify_scan:
  extends: .fortify_scan_base   # inherits all config from .fortify_scan_base
  stage: scan
  only:
    - main
    - develop
    - /^release\/.*/
```

The leading dot in `.fortify_scan_base` is GitLab's convention for a **hidden job** — it defines reusable config but won't run on its own. The `extends:` keyword in your `fortify_scan` job pulls everything in (script, tags, artifacts) and you can override individual fields if needed.

**Why keep it as a separate file rather than pasting it into `.gitlab-ci.yml`?**

The main reason is reuse. If you have multiple microservice repos all needing the same Fortify scan, each one's `.gitlab-ci.yml` can include this file rather than duplicating 200 lines of PowerShell. When you need to update the scan logic (e.g. new Fortify version, new SSC URL), you change it in one place.

You can take this further by hosting the template in a **central DevSecOps repo** and including it across the whole organisation:

```yaml
# In any project's .gitlab-ci.yml — no local copy needed
include:
  - project: 'devsecops/pipeline-templates'
    ref: main
    file: '/fortify_scan.yml'
```

GitLab fetches the file from that repo at pipeline runtime. Every project automatically picks up updates on their next run.

---

## Adapting for GitHub Actions

The core PowerShell script is identical — only the variable names and job structure change. Here is the GitHub Actions equivalent:

```yaml
# .github/workflows/fortify-scan.yml

name: Fortify SAST Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  fortify-scan:
    runs-on: [self-hosted, windows, fortify]  # Self-hosted Windows runner with Fortify installed

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run Fortify SAST Scan
        shell: pwsh
        env:
          FORTIFY_SSC_URL: ${{ secrets.FORTIFY_SSC_URL }}
          FORTIFY_TOKEN:   ${{ secrets.FORTIFY_TOKEN }}
          # Map GitHub context variables to match what the script expects
          CI_PROJECT_NAME:      ${{ github.event.repository.name }}
          CI_COMMIT_REF_NAME:   ${{ github.ref_name }}
        run: |
          # Paste the script block from fortify_scan.yml here
          # Everything from [Net.ServicePointManager] onwards works unchanged

      - name: Upload FPR artifact
        uses: actions/upload-artifact@v3
        if: success()
        with:
          name: fortify-results
          path: "*.fpr"
          retention-days: 7
```

**Key differences between GitLab CI and GitHub Actions:**

| | GitLab CI | GitHub Actions |
|---|---|---|
| Project name | `$env:CI_PROJECT_NAME` | `${{ github.event.repository.name }}` |
| Branch name | `$env:CI_COMMIT_REF_NAME` | `${{ github.ref_name }}` |
| Secret reference | `$env:FORTIFY_TOKEN` | `${{ secrets.FORTIFY_TOKEN }}` (passed via `env:`) |
| Runner tag | `tags: [fortify-slave-1]` | `runs-on: [self-hosted, windows, fortify]` |
| Artifacts | `artifacts: paths:` | `actions/upload-artifact` |

The PowerShell script body between those two is **copy-paste identical** — no changes needed.

---

## SSC app naming convention

| Field | Value | Example |
|---|---|---|
| App Name | `{project-name}-{branch-name}` | `payment-service-main` |
| App Version | `{branch-name}` | `main`, `feature-login-fix` |

Special characters in project and branch names are replaced with hyphens automatically. `Credit_Cards` becomes `credit-cards`, `Feature/Login-Fix` becomes `feature-login-fix`.

---

## Changing the issue template

The script uses `Prioritized-HighRisk-Project-Template` by default — the standard Fortify template that prioritises Critical and High findings. If your SSC admin has configured a custom template, update this line:

```powershell
$templateId = "Prioritized-HighRisk-Project-Template"
```

You can find available template names in SSC under **Administration → Issue Templates**.

---

## Troubleshooting

**`FPR file was not created`** — The scan failed silently. Check the `sourceanalyzer.exe` output above this message in the job log. Common causes: unsupported language, missing JDK for Java projects, or insufficient disk space on the runner.

**`fortifyclient upload failed (exit code: 1)`** — Usually an auth or network issue. Verify `FORTIFY_SSC_URL` doesn't have a trailing slash, and that `FORTIFY_TOKEN` hasn't expired in SSC.

**`400` from SSC version creation** — The version may already exist with a slightly different name. Check the SSC application list and compare with the `App Name` printed at the start of the job log.

**TLS errors** — The `[Net.ServicePointManager]::SecurityProtocol` line forces TLS 1.2. If you're still seeing TLS errors, your runner's .NET version may be too old. Update .NET Framework on the runner to 4.6+.

---

## Requirements summary

- GitLab CI/CD (or GitHub Actions with a self-hosted runner — see above)
- Windows runner with Fortify SCA installed
- Fortify SSC accessible from the runner
- PowerShell 5.1+ (standard on Windows Server 2016+)
- .NET Framework 4.6+ on the runner (for TLS 1.2 support)
