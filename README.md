# fab-test Playwright templates

Starter files for running [`fab-test playwright`](https://github.com/clientfirsttech/fab-test) against a Microsoft Fabric workspace. It opens every deployed report in a browser, page by page, and fails the run when a visual does not render.

| File | Purpose |
|------|---------|
| [.github/workflows/playwright-check.yml](.github/workflows/playwright-check.yml) | GitHub Actions workflow using Chromium on the runner |
| [.github/workflows/playwright-checks-azure.yml](.github/workflows/playwright-checks-azure.yml) | GitHub Actions workflow using Azure App Testing (Playwright Workspaces) browsers |
| [.fab-test/playwright.yml](.fab-test/playwright.yml) | Execution config that selects the Azure browser backend. Only used when passed with `--playwright-config` (the Azure workflow does this). Plain laptop runs and the runner workflow ignore it |

`fab-test init` creates `fab-test.yml`, `.fab-test/.env.example`, and `.fab-test/.gitignore` for you (see [Step 5](#step-5-run-it-locally)).

The full walkthrough lives in the fab-test docs: [PLAYWRIGHT-CI.md](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md). This README summarizes the order and links to each step.

The three ways to run, in the order to adopt them:

1. [On your laptop](#part-1-on-your-laptop), with a local browser. Proves the setup.
2. [In GitHub Actions](#part-2-github-actions-browsers-on-the-runner), with Chromium on the runner.
3. [In GitHub Actions with Azure App Testing](#part-3-github-actions-with-azure-app-testing), with browsers hosted by [Azure App Testing (Playwright Workspaces)](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/overview-what-is-microsoft-playwright-workspaces).

## Part 1: On your laptop

The laptop run uses a local Chromium and needs no execution config file. Complete the Fabric and Entra setup below first. Do these in order. Each step is a prerequisite for the next, and the local run (step 5) proves the setup before CI is involved.

| Step | Where | Who | Docs |
|------|-------|-----|------|
| 1. Register a service principal | Entra ID | Entra app admin | [Register the service principal](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#1-register-the-service-principal) |
| 2. Allow service principals in the tenant | Fabric admin portal | Fabric admin | [Tenant settings](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#2-allow-service-principals-in-the-tenant) |
| 3. Give it a workspace role, enable XMLA | Fabric workspace and capacity | Workspace and capacity admins | [Workspace role](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#3-give-it-a-workspace-role) |
| 4. Deploy the reports to test | Fabric workspace | You | [Deploy reports](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#4-deploy-the-reports-you-want-to-test) |
| 5. Run it locally | Your machine | You | [Run it locally](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#5-run-it-locally) |

Prefer the Azure CLI to the portal? [GETTING-STARTED.md](https://github.com/clientfirsttech/fab-test/blob/main/docs/GETTING-STARTED.md) scripts steps 1 and 3 with `az`.

`fab-test playwright` is the one analyzer that cannot fall back to `az login`. It needs a full service principal to generate Power BI embed tokens.

### Quick summary of steps 1 to 3

- **Service principal:** add the Power BI Service API permissions `App.Read.All`, `Dataset.Read.All`, `SemanticModel.Read.All`, `Report.Read.All`, and `Workspace.Read.All`, then grant admin consent. Create a client secret and put the service principal in an Entra security group.
- **Tenant settings** (Developer settings, scoped to that group): *Service principals can use Fabric APIs* and *Embed content in apps*. Add *Dataset Execute Queries REST API* if you test paginated reports with parameters. Changes can take about 15 minutes.
- **Workspace:** add the service principal (or its group) with the **Member** role. Viewer is not enough. The workspace must be on a capacity with the XMLA endpoint enabled.

### Step 5: run it locally

From PowerShell in the repo root, create and activate a virtual environment, then install fab-test with the `playwright` extra and the browser:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1

pip install --pre "cft-fab-test[playwright]"
playwright install chromium

fab-test init
Copy-Item .fab-test\.env.example .fab-test\.env
```

Run `.\.venv\Scripts\Activate.ps1` again in every new PowerShell session before using `fab-test`.

Fill in `.fab-test/.env`. The file is gitignored, never commit it.

```dotenv
FABRIC_TENANT_ID=<directory (tenant) id>
FABRIC_CLIENT_ID=<application (client) id>
FABRIC_CLIENT_SECRET=<client secret value>

# Only needed to test row-level security
# PLAYWRIGHT_USER_NAME=analyst@contoso.com
```

Then check readiness and run:

```powershell
fab-test doctor
fab-test playwright --workspace "<workspace name or GUID>" --report
```

A bare `--workspace` tests every deployed report and paginated report in that workspace, with no checkout of the reports needed. Add `--artifact "Report Name"` to narrow it, or `--plan-only` to see the test matrix without launching a browser. Add `--headed` to watch the browser, and `--workers N` to set how many cases run at once.

To customize local browsers from a file, use an execution YAML with `backend: local` and pass it with `--playwright-config` (see the upstream [headed.yml](https://github.com/clientfirsttech/fab-test/blob/main/docs/examples/playwright/headed.yml)). [.fab-test/playwright.yml](.fab-test/playwright.yml) in this repo is set to `backend: azure`, so passing it from your laptop sends the browsers to [Azure App Testing](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/overview-what-is-microsoft-playwright-workspaces) instead. See [Part 3](#part-3-github-actions-with-azure-app-testing) for that setup.

Reference material:

- [Credentials resolution and `fab-test auth status`](https://github.com/clientfirsttech/fab-test/blob/main/.github/skills/fab-test/references/credentials.md)
- [All `playwright` flags](https://github.com/clientfirsttech/fab-test/blob/main/.github/skills/fab-test/references/flags.md)
- [`fab-test.yml` settings and precedence](https://github.com/clientfirsttech/fab-test/blob/main/.github/skills/fab-test/references/configuration.md)
- [Exit codes](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#exit-codes) and [troubleshooting](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#troubleshooting)

> **Beta release:** fab-test is currently a beta. pip skips pre-releases by default, so install with `--pre`. The version is intentionally not pinned, so new beta builds are picked up automatically.

## Part 2: GitHub Actions (browsers on the runner)

Once the laptop run passes, run the same check in CI with [playwright-check.yml](.github/workflows/playwright-check.yml). The runner hosts Chromium itself, so nothing is needed beyond GitHub and Fabric. When runner CPU becomes the limit, move to [Part 3](#part-3-github-actions-with-azure-app-testing), which offloads the browsers to [Azure App Testing](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/overview-what-is-microsoft-playwright-workspaces).

The workflow is manual only (`workflow_dispatch`). It never runs on `pull_request`, because a fork PR has no access to repository secrets. Inputs:

| Input | Required | Purpose |
|-------|----------|---------|
| `workspace` | Yes | Fabric workspace name or GUID |
| `artifact` | No | Report name to narrow the run. Blank tests every deployed report |
| `environment` | No | Label only, such as DEV, TEST, or PROD |

### What the workflow does

1. Checks out the repo, installs Python 3.12, `cft-fab-test[playwright]` (with `--pre`), and Chromium.
2. Runs `fab-test auth status` as a diagnostic. It uses `continue-on-error` because it checks the Fabric API scope, not the embed-token path Playwright uses.
3. Runs `fab-test playwright --workspace ... --report`, adding `--artifact` and `--env` only when those inputs are filled in.
4. Uploads `fab-test-results/` even on failure.
5. Writes a pass/fail table and the first 6 failing screenshots (inline) to the job summary.

### Required GitHub secrets

Add these under **Settings > Secrets and variables > Actions** (or in a GitHub Environment, see below).

| Name | Type | Value |
|------|------|-------|
| `FABRIC_TENANT_ID` | Secret | Directory (tenant) ID |
| `FABRIC_CLIENT_ID` | Secret | Application (client) ID |
| `FABRIC_CLIENT_SECRET` | Secret | Client secret value |

Optional additions to the workflow `env:` block:

| Name | Purpose |
|------|---------|
| `PLAYWRIGHT_USER_NAME` | Effective-identity UPN. Required to test reports on models with row-level security. Without it, discovered roles abort the run before any token is minted. |
| `PLAYWRIGHT_TIMEOUT_SECONDS` | Raise from the default of 180 for slow pages or large models. |

Other ways to supply the workspace, if you do not want to type it at dispatch time:

- Set `FABRIC_WORKSPACE_ID` as an Actions **variable** (a workspace ID is metadata, not a credential).
- Set `workspace:` in `fab-test.yml` (created by `fab-test init`).

A workspace set that way is only a default. A bare `fab-test playwright` run still needs `--workspace` (as both workflows pass) to test the whole workspace without a checkout.

#### Optional: use a GitHub Environment

To require reviewer approval before a run touches a real workspace, put the secrets in an Environment and add `environment: <name>` to the job. The fab-test example workflows use an Environment named `fabric-demo`. See [Create the GitHub Environment](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#create-the-github-environment) and the upstream examples:

- [playwright-live.yml](https://github.com/clientfirsttech/fab-test/blob/main/docs/examples/github-actions/playwright-live.yml) (runner browsers)
- [playwright-azure.yml](https://github.com/clientfirsttech/fab-test/blob/main/docs/examples/github-actions/playwright-azure.yml) (Azure browsers, see Part 3)

### Template edits before first use

- Replace the `default: "<your-workspace-name>"` workspace input with your own workspace, or remove the default.
- Make sure the `pip install` step uses `--pre`: `pip install --pre "cft-fab-test[playwright]"`.
- Optionally add `PLAYWRIGHT_USER_NAME` to the `env:` block if your models use row-level security.

## Part 3: GitHub Actions with Azure App Testing

This variant runs the same checks, but the browsers are hosted by [Azure App Testing (Playwright Workspaces)](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/overview-what-is-microsoft-playwright-workspaces) instead of the runner. Python, pytest, and xdist still run on the runner. Use it when raising `jobs` on the runner stops shortening the wall clock. See the fab-test [timing comparison](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#reading-run-timing-is-jobs-or-azure-worth-it). New to the service? Microsoft offers a [free trial](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/how-to-try-playwright-workspaces-free).

| | [playwright-check.yml](.github/workflows/playwright-check.yml) (Part 2) | [playwright-checks-azure.yml](.github/workflows/playwright-checks-azure.yml) (Part 3) |
|---|---|---|
| Where browsers run | Chromium on the GitHub runner | Azure App Testing (Playwright Workspaces) |
| Installs Chromium | Yes (`playwright install --with-deps chromium`) | No |
| Execution config | None, local default | `--playwright-config .fab-test/playwright.yml` |
| Extra input | `environment` (label only) | None |
| Secrets needed | 3 Fabric secrets | 3 Fabric secrets plus 2 Azure secrets |
| Results artifact | `fab-test-results` | `fab-test-results-azure` (14 day retention) |
| Step summary | Table plus up to 6 inline failure screenshots | Table only |
| Concurrency group | `fab-test-playwright-<workspace>` | `fab-test-playwright-azure-<workspace>` |

### Set up Azure App Testing

fab-test connects with a **service access token**, not Entra ID, so token authentication must be enabled on the workspace. Microsoft's own guidance recommends Entra ID where possible, see [Playwright Workspaces authentication](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/how-to-manage-authentication).

1. **Create a Playwright workspace.** In the Azure portal, create a resource, search for *Playwright Workspaces*, then choose a subscription, resource group, name, and location. You need Owner or Contributor on the subscription. Details: [Quickstart: Run Playwright tests at scale](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/quickstart-run-end-to-end-tests).
2. **Enable access token authentication.** It is off by default. Go to the workspace, **Settings > Access Management**, and check **Playwright Service Access Token**. Details: [Enable authentication using access tokens](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/how-to-manage-access-tokens#enable-authentication-using-access-tokens).
3. **Copy the browser endpoint.** On the workspace **Get Started** page, copy the endpoint URL. It looks like `wss://<region>.api.playwright.microsoft.com/playwrightworkspaces/<workspace-id>/browsers`. This becomes `PLAYWRIGHT_SERVICE_URL`.
4. **Generate an access token.** On **Access Management**, choose **Generate token**, set an expiration, and copy the value immediately. It cannot be retrieved later. This becomes `PLAYWRIGHT_SERVICE_ACCESS_TOKEN`. Details: [Generate a workspace access token](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/how-to-manage-access-tokens#generate-a-workspace-access-token).
5. **Store the values.**
   - Locally: put both in `.fab-test/.env` (gitignored).
   - In CI: add both as GitHub secrets named `PLAYWRIGHT_SERVICE_URL` and `PLAYWRIGHT_SERVICE_ACCESS_TOKEN`.
6. **Verify from your laptop** (activate the virtual environment first with `.\.venv\Scripts\Activate.ps1`):

   ```powershell
   fab-test playwright --workspace "<workspace>" --playwright-config .fab-test/playwright.yml --report
   ```

   Run `fab-test doctor` first. It checks the Azure values without connecting.

The Fabric service principal and the Azure token are separate credentials. The `FABRIC_*` values mint Power BI embed tokens, and the Azure token only authorizes the browser service. Neither substitutes for the other.

### Workflow: playwright-checks-azure.yml

It is manual only (`workflow_dispatch`) and takes `workspace` (required) and `artifact` (optional) inputs.

1. Checks out the repo, installs Python 3.12 and `cft-fab-test[playwright]`. No browser download is needed.
2. Runs `fab-test playwright` with `--playwright-config .fab-test/playwright.yml`, which sends the browsers to Azure. Verdicts and evidence are the same as Part 2.
3. Uploads `fab-test-results/` as `fab-test-results-azure`.
4. Writes a pass/fail table to the job summary.

A connection or authentication failure to Azure is reported as an execution error (exit `1`), never as a broken-visual finding, and there is no silent fallback to local browsers.

### Additional GitHub secrets

Add these on top of the three Fabric secrets from Part 2.

| Name | Type | Value |
|------|------|-------|
| `PLAYWRIGHT_SERVICE_URL` | Secret | Regional `wss://` browser endpoint |
| `PLAYWRIGHT_SERVICE_ACCESS_TOKEN` | Secret | Playwright workspace access token |

### Template edits before first use

- Replace the `default: "<your-workspace-name>"` workspace input with your own workspace, or remove the default.
- Make sure the `pip install` step uses `--pre`: `pip install --pre "cft-fab-test[playwright]"`.
- Commit [.fab-test/playwright.yml](.fab-test/playwright.yml) with your settings. It holds no credentials.

### Execution config

[.fab-test/playwright.yml](.fab-test/playwright.yml) controls how the Azure run is shaped. It only takes effect when passed with `--playwright-config`, and it does not affect plain laptop runs or the Part 2 workflow:

```yaml
backend: azure
workers: 8          # parallel cases within one report
jobs: 4             # reports run at once (up to jobs x workers browsers)
connection:
  os: linux
  timeout_ms: 30000
  expose_network: "<loopback>"
context:
  viewport:
    width: 1280
    height: 720
```

- Keep this file free of tokens and credential-bearing URLs. fab-test rejects anything beyond the supported keys.
- Start with a modest `jobs` and `workers` and stay within the [Azure App Testing limits and quotas](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/resource-limits-quotas-capacity). More workers do not always mean faster reports, since each report's workers are capped by its case count.
- `--headed` and `--slow-mo` are ignored with a warning on Azure browsers.
- Playwright Workspaces bills by test minutes. Try a single report with `--artifact` before running a whole workspace.

### Token hygiene

Access tokens behave like long-lived passwords.

- Set a short expiration and rotate before it lapses. Update the local `.env` and the GitHub secret together.
- Never put the token in YAML, workflow files, or the command line.
- Revoke and regenerate immediately if it is exposed.
- Microsoft recommends Entra ID where possible, but this fab-test release supports access tokens only for the browser service. See [Manage workspace access tokens](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/how-to-manage-access-tokens) and fab-test's [Azure browser setup](https://github.com/clientfirsttech/fab-test/blob/main/docs/PLAYWRIGHT-CI.md#azure-hosted-browsers).

## Reading the results

Each run writes to `fab-test-results/` (gitignored), and the workflows upload it as an artifact.

| Path | Contents |
|------|----------|
| `run.json` | Run manifest: exit code, totals, per-report status |
| `index.html` | One page linking every report's result (with `--report`) |
| `playwright/<report>/envelope.json` | Per-case results and a link to the page tested |
| `playwright/<report>/report.html` | Readable report with screenshots |
| `playwright/<report>/test-cases/<case>/` | `screenshot.png`, plus `console.json`, `network.json`, `event_log.json` when relevant |

Exit codes: `0` all cases rendered, `1` a case failed or the run errored, `2` invalid arguments or config, `127` missing credentials or prerequisites.
