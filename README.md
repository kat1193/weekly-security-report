# Weekly Security Requirements Report — GitHub Actions Automation Guide

This sets up a fully unattended job that runs every Friday, queries Azure
DevOps for `[SEC-REQ]`-tagged work items, and updates the slide 10 chart in
`Weekly_Technical_Status_Report.pptx` — no local machine, no open session,
no expiring cron job.

---

## Why this approach (vs. what we tried before)

| | Local `CronCreate` | Cloud Routine | **GitHub Actions** |
|---|---|---|---|
| Runs without your machine on | ❌ | ✅ | ✅ |
| Doesn't expire after 7 days | ❌ | ✅ | ✅ |
| Has a real encrypted secrets store | N/A (local file access) | ❌ (not yet available for cloud sessions) | ✅ |
| Reaches Azure DevOps | ✅ (local MCP) | ❌ (local MCP unreachable, no hosted MCP auth yet) | ✅ (direct REST API call) |
| PPTX location | Local `~/Downloads` | Needs cloud-reachable storage | Lives in the repo itself |

GitHub Actions is the only option that's actually "set it up once, runs
forever, nothing exposed insecurely."

---

## Step 1: Get the repo ready

The PPTX needs to live in a GitHub repository so the workflow can check it
out, edit it, and commit the result back.

- If you don't have one yet, create a new repo (can be private) and add
  `Weekly_Technical_Status_Report.pptx` to it.
- If it already lives in a repo, just work from there.

---

## Step 2: Create the Azure DevOps Personal Access Token (PAT)

This is the credential that lets the workflow authenticate to Azure DevOps.

1. Go to `https://dev.azure.com/{your-org}`
2. Click your user icon (top right) → **Personal access tokens**
3. Click **+ New Token**
4. Name it something identifiable, e.g. `github-actions-weekly-report`
5. Under **Scopes**, select **Work Items → Read** (read-only — this
   automation only needs to query, not modify, work items)
6. Set an expiration date (note it somewhere — you'll need to regenerate and
   update the secret before it expires)
7. Click **Create**, then **copy the token immediately** — Azure DevOps only
   displays it once

---

## Step 3: Get a Claude Code credential (subscription OAuth token)

Used a Claude Pro/Max subscription instead of a separate Anthropic Console
API key — this authenticates headless Claude Code against the existing
subscription rather than a pay-as-you-go billing account.

1. On a local machine, with Claude Code installed and logged into the
   Pro/Max account:
   ```bash
   claude setup-token
   ```
2. This opens a browser auth flow and prints a long-lived token
   (`sk-ant-oat01-...`) — copied for the next step.

Note: subscriptions include separate monthly credits for this kind of
automation usage (covers `claude -p` / GitHub Actions runs), refreshed
monthly. A weekly job like this one is light usage and should sit
comfortably within that allowance.

---

## Step 4: Add both as GitHub repository secrets

Secrets are encrypted at rest and only decrypted inside the running job —
this is what keeps the PAT and token safe.

1. In the repo on GitHub: **Settings → Secrets and variables → Actions**
2. **New repository secret**
   - Name: `ADO_PAT` → value: the Azure DevOps token from Step 2
3. **New repository secret** again
   - Name: `CLAUDE_CODE_OAUTH_TOKEN` → value: the token from Step 3

Neither of these should ever go directly in the workflow file or be
committed anywhere in the repo — only in the Secrets settings.

---

## Step 5: Add the workflow file

Create `.github/workflows/weekly-report.yml` in the repo:

```yaml
name: Weekly Security Requirements Report

on:
  schedule:
    - cron: '0 15 * * 5'   # 5pm Athens time (UTC+2) on Fridays — cron runs in UTC
  workflow_dispatch: {}     # allows manual runs for testing

permissions:
  contents: write   # required so the final step can push the commit back

jobs:
  update-report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # This step runs as a plain shell step on the runner — NOT through
      # Claude's bash tool. The PAT is only ever touched here, never by the
      # agent, so Claude Code's secret-expansion guardrail never comes into
      # play. Two separate WIQL queries give us To Do vs Done counts.
      # NOTE: [SEC-REQ] is a prefix in the work item TITLE, not a tag —
      # queries filter on System.Title, not System.Tags.
      - name: Query Azure DevOps for [SEC-REQ] work items
        env:
          ADO_PAT: ${{ secrets.ADO_PAT }}
        run: |
          curl -s --globoff -u ":$ADO_PAT" \
            -H "Content-Type: application/json" \
            -w "\nHTTP status: %{http_code}\n" \
            -d '{"query": "SELECT [System.Id] FROM WorkItems WHERE [System.Title] CONTAINS \"SEC-REQ\" AND [System.State] IN (\"To Do\",\"Doing\")"}' \
            "https://dev.azure.com/testbedcns/PRBN3336/_apis/wit/wiql?api-version=7.1" \
            -o todo-items.json

          curl -s --globoff -u ":$ADO_PAT" \
            -H "Content-Type: application/json" \
            -w "\nHTTP status: %{http_code}\n" \
            -d '{"query": "SELECT [System.Id] FROM WorkItems WHERE [System.Title] CONTAINS \"SEC-REQ\" AND [System.State] = \"Done\""}' \
            "https://dev.azure.com/testbedcns/PRBN3336/_apis/wit/wiql?api-version=7.1" \
            -o done-items.json

          echo "--- todo-items.json ---" && cat todo-items.json
          echo "--- done-items.json ---" && cat done-items.json

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      # Claude only ever sees the two JSON files above — never the PAT,
      # never a curl command containing it.
      # --dangerously-skip-permissions is safe specifically here: this is a
      # disposable, isolated CI container with nothing beyond this repo
      # checkout — not a flag to use on a real machine or a session with
      # broader filesystem/credential access.
      - name: Update pptx from queried data
        env:
          CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
        run: |
          claude -p --dangerously-skip-permissions "Read todo-items.json and done-items.json in this repo
          — each contains an Azure DevOps WIQL result with a workItems
          array. Count the items in each. Open
          Weekly_Technical_Status_Report.pptx and update slide 10's
          'BRD Analysis' mini-chart specifically (not the Design/Technical
          chart) — it has two series: 'Business Security Requirements
          Identified' and 'Business Security Requirements Implemented'. Set
          the todo-items.json count as 'Business Security Requirements
          Identified' and the done-items.json count as 'Business Security
          Requirements Implemented'. Save the file in place."

      - name: Commit updated PPTX
        run: |
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git add Weekly_Technical_Status_Report.pptx
          git commit -m "Weekly security requirements update" || echo "No changes"
          git push
```

**Why this shape matters:** the PAT is only ever expanded inside a runner
shell step written by you (checked into the repo, reviewed once), never
inside a command Claude constructs at runtime. That's not just a workaround
for the guardrail — it's the correct security boundary: an agent should
operate on data, not on the credential that fetched it.

**Placeholders are now filled in** with the real org (`testbedcns`) and
project (`PRBN3336`). Two things still worth checking:
- Double check the cron time. `0 15 * * 5` assumes Athens is UTC+2 (CEST,
  summer). Athens shifts to UTC+2/UTC+3 with daylight saving — recheck this
  twice a year, or use a cron-timezone-aware tool if your GitHub plan
  supports it.
- The `-w "\nHTTP status: %{http_code}\n"` and `cat` lines print the raw
  API response into the Actions log — useful for debugging now, but it also
  means work item IDs/titles will be visible in the (private) repo's Action
  logs. Fine for a private repo; remove those two debug lines once you've
  confirmed everything works, if you'd rather not have that in the logs
  long-term.

---

## Step 6: Test it manually before trusting the schedule

1. Commit and push the workflow file
2. Go to the repo's **Actions** tab → select **Weekly Security Requirements
   Report** → **Run workflow** (this uses the `workflow_dispatch` trigger)
3. Watch the run, then check:
   - Did it correctly query `[SEC-REQ]` items by state?
   - Did slide 10's chart update with sensible numbers?
   - Did the commit land with the updated file?

Only once this looks right should you leave it to fire on the Friday
schedule unattended.

---

## Step 7: Ongoing maintenance

- **PAT expiration**: when the Azure DevOps token expires, regenerate it
  (Step 2) and update the `ADO_PAT` secret (Step 4) — the workflow will
  otherwise start failing silently on Fridays.
- **Check Action runs occasionally**: GitHub emails you on workflow failures
  by default, but it's worth glancing at the Actions tab periodically.

---

## Readiness checklist — is it actually ready to run?

Go through this before trusting the Friday schedule. All five should be
true:

- [ ] **Repo has the file**: `Weekly_Technical_Status_Report.pptx` is
  committed and visible in the repo (not just sitting locally).
- [ ] **Workflow file is committed**: `.github/workflows/weekly-report.yml`
  is pushed to the default branch (usually `main`). Check the repo's
  **Actions** tab — a workflow named **Weekly Security Requirements Report**
  should be listed there. If it's not listed, the file either didn't get
  pushed or has a YAML syntax error.
- [ ] **Both secrets exist**: **Settings → Secrets and variables → Actions**
  should show `ADO_PAT` and `CLAUDE_CODE_OAUTH_TOKEN` listed (values are
  hidden, but the names should appear).
- [ ] **Org/project are correct**: `testbedcns` / `PRBN3336` are already
  filled into both curl URLs — just confirm these are still the right org
  and project if anything changes later.
- [ ] **Cron time is correct for your timezone right now**: `0 15 * * 5`
  assumes Athens at UTC+2. Double check this is still right for the current
  time of year (Greece observes daylight saving).

### How to actually test it (don't wait for Friday)

1. Go to the repo's **Actions** tab
2. Click **Weekly Security Requirements Report** in the left sidebar
3. Click **Run workflow** (this uses the `workflow_dispatch` trigger already
   in the file) → **Run workflow** again to confirm
4. Watch the run — click into it to see live logs for each step
   (checkout → install → run report update → commit)
5. If it goes green end to end:
   - Open `Weekly_Technical_Status_Report.pptx` in the repo and check slide
     10's chart actually updated with numbers that look right
   - Check the commit history — there should be a new commit "Weekly
     security requirements update"
6. If a step fails (red ❌), click it to expand the logs — common failure
   points:
   - **Install Claude Code step fails**: usually a transient npm issue, just
     re-run
   - **Run report update fails with an auth error**: check the `ADO_PAT` or
     `CLAUDE_CODE_OAUTH_TOKEN` secret values are correct and not expired
   - **Runs but chart doesn't update**: check that `[SEC-REQ]` still matches
     how it actually appears in the work item title in Azure DevOps (e.g. if
     someone starts writing it as `SEC REQ` without the hyphen, `CONTAINS`
     won't match) — check the `cat todo-items.json` /
     `cat done-items.json` output in the Actions log to confirm real data
     came back, not an empty `workItems` array or an error response.
   - **`403` / "Write access to repository not granted" on the commit step**:
     the workflow needs `permissions: contents: write` — scheduled workflows
     get a read-only token by default, so pushing back fails without it
     (already included in the workflow above; if you're on an older copy of
     the file, add it under the `on:` block).

Once a manual run succeeds and the file looks right, the Friday schedule
will fire the same way automatically — no further action needed until the
PAT expires.
