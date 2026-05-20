---
description: Triage all open CVEs for OpenShift Node team components with reachability analysis
argument-hint: "[--component <name>] [--notify-jira] [--notify-slack] [--days N]"
---

## Name
node-cve:triage

## Synopsis
```text
/node-cve:triage [--component "Node / CRI-O"] [--notify-jira] [--notify-slack] [--days 7]
```

## Description

Queries all open CVE vulnerability issues in OCPBUGS for Node team components, deduplicates across version trackers, clones affected repositories, and analyzes source code for CVE reachability. Optionally posts analysis comments to Jira trackers and sends a Slack summary.

Designed for both interactive use and headless daily execution via `claude --print`.

## Prerequisites

- `jira` CLI ([ankitpokhrel/jira-cli](https://github.com/ankitpokhrel/jira-cli)) configured with Jira credentials
- Environment variables: `JIRA_API_TOKEN`, `JIRA_USERNAME`
- `git` (for cloning repos)
- Optional: `curl` and `SLACK_WEBHOOK` environment variable (for `--notify-slack`)

## Implementation

### Phase 0: Setup and Argument Parsing

1. **Parse Arguments**
   - `--component <name>`: Filter to a specific OCPBUGS component (e.g., "Node / CRI-O"). Optional.
   - `--notify-jira`: Post analysis results as comments on Jira tracker issues. Default: off.
   - `--notify-slack`: Send summary to Slack webhook. Requires `$SLACK_WEBHOOK`. Default: off.
   - `--days N`: Only include CVEs created or updated in the last N days. Default: all open.

2. **Validate Tools**

   ```bash
   which jira 2>/dev/null || echo "MISSING: jira CLI"
   which git 2>/dev/null || echo "MISSING: git"
   ```

   If any required tool is missing, display installation instructions and exit.

3. **If `--notify-slack`**, verify `$SLACK_WEBHOOK` is set. Exit with error if missing.

4. **Create work directory**: `mkdir -p .work/node-cve/repos .work/node-cve/triage-$(date +%Y-%m-%d)`

---

### Phase 1: Query Open CVEs

- **Skill**: [query-open-cves](../skills/query-open-cves/SKILL.md)
- **Input**: Optional `--component` filter, optional `--days` filter
- **Output**: Deduplicated list of CVEs with metadata

**Steps:**

1. Build the JQL query using OCPBUGS component names from the Node team in `team_component_map.json`:

   ```bash
   jira issue list -q "project = OCPBUGS AND type = Vulnerability AND component in (\"Node / CRI-O\", \"Node / Kubelet\", \"Node / CPU manager\", \"Node / Device Manage\", \"Node / Memory manager\", \"Node / Numa aware Scheduling\", \"Node / Pod resource API\", \"Node / Topology manager\", \"Driver Toolkit\", \"Machine Config Operator\") AND status not in (Closed, Done, Verified)" --plain --no-headers --columns KEY,SUMMARY,STATUS,ASSIGNEE,LABELS
   ```

   If `--component` is specified, replace the component list with the single component.
   If `--days N` is specified, add `AND updated >= -${N}d` to the query.

2. Parse results and extract CVE IDs from summaries (regex: `CVE-[0-9]{4}-[0-9]+`).

3. Deduplicate: group tracker issues by CVE ID. For each unique CVE, collect:
   - All tracker keys (e.g., OCPBUGS-85948, OCPBUGS-85932, ...)
   - Affected OCP versions (extracted from summary brackets, e.g., `[openshift-4.19]`)
   - Component name
   - Assignee (from the highest version tracker)
   - Status

4. Print intermediate summary: "Found N unique CVEs across M tracker issues."

**Decision Point:**
- IF 0 CVEs found -> Print "No open CVEs for Node team components." -> Exit
- IF CVEs found -> Continue to Phase 2

---

### Phase 2: Analyze CVEs Against Affected Repos

- **Skill**: [analyze-cve-repos](../skills/analyze-cve-repos/SKILL.md)
- **Input**: List of unique CVEs from Phase 1
- **Output**: Reachability results per CVE

For each unique CVE:

1. **Determine affected repo** from the OCPBUGS component:

   | Component | Repository | Language |
   |-----------|-----------|----------|
   | Node / CRI-O | https://github.com/cri-o/cri-o | Go |
   | Node / Kubelet | https://github.com/openshift/kubernetes | Go |
   | Node / CPU manager | https://github.com/openshift/kubernetes | Go |
   | Node / Device Manage | https://github.com/openshift/kubernetes | Go |
   | Node / Memory manager | https://github.com/openshift/kubernetes | Go |
   | Node / Numa aware Scheduling | https://github.com/openshift/kubernetes | Go |
   | Node / Pod resource API | https://github.com/openshift/kubernetes | Go |
   | Node / Topology manager | https://github.com/openshift/kubernetes | Go |
   | Driver Toolkit | https://github.com/openshift/driver-toolkit | Go |
   | Machine Config Operator | https://github.com/openshift/machine-config-operator | Go |

   Also check labels for `pscomponent:conmon`, `pscomponent:conmon-rs`, or `pscomponent:cri-tools` and map:
   - conmon -> https://github.com/containers/conmon (C)
   - conmon-rs -> https://github.com/containers/conmon-rs (Rust + Go)
   - cri-tools -> https://github.com/kubernetes-sigs/cri-tools (Go)

2. **Clone the repo** (if not already cloned in this run):

   ```bash
   git clone --depth 1 <repo-url> .work/node-cve/repos/<repo-name>/
   ```

   Use `--depth 1` for speed.

3. **Gather CVE intelligence**: fetch vulnerability details from public sources (NVD, advisories, Jira issue description/comments) to identify the affected package, vulnerable functions, and attack vector.

4. **Check dependency presence**: search dependency files (`go.mod`, `Cargo.toml`, build files, vendored code) for the affected package. If not present, classify as NOT_AFFECTED and continue to next CVE.

5. **Analyze source code for reachability**: search the codebase for imports and calls to the vulnerable function. Read the relevant source files to trace call paths from entry points to the vulnerable function. Assess whether attacker-controlled input reaches the vulnerable code path and identify any mitigating controls (input validation, size limits, feature flags, authentication).

6. **Classify result**:
   - **REACHABLE** (HIGH): vulnerable function called with attacker-controlled input, no mitigations
   - **REACHABLE** (MEDIUM): vulnerable function called, but input is partially validated
   - **PRESENT_NOT_EXPLOITABLE** (HIGH): vulnerable function called, but only with trusted/internal data
   - **PRESENT_NOT_REACHABLE** (HIGH): package imported but vulnerable function not called
   - **NOT_AFFECTED** (HIGH): package not in dependency tree
   - **UNCERTAIN** (LOW): repo too large to fully analyze, or CVE details insufficient

7. **Save analysis** to `.work/node-cve/triage-$(date +%Y-%m-%d)/<CVE-ID>-analysis.md` with file paths, call sites, and evidence.

8. Print progress: "Analyzed CVE-XXXX-XXXXX against <repo>: <result>"

---

### Phase 3: Report Findings

- **Skill**: [report-findings](../skills/report-findings/SKILL.md)
- **Input**: Analysis results from Phase 2, notification flags
- **Output**: Report file, optional Jira comments, optional Slack message

1. **Generate report** at `.work/node-cve/triage-$(date +%Y-%m-%d)/report.md`:

   ```markdown
   # Node CVE Triage Report - YYYY-MM-DD

   ## Summary
   - Total unique CVEs: N
   - Reachable: N (list)
   - Present but not reachable: N (list)
   - Not affected: N (list)
   - Uncertain: N (list)
   - Unassigned CVEs: N
   - CVEs with due dates in next 7 days: N

   ## Detailed Findings

   ### CVE-XXXX-XXXXX: <description>
   - **Component**: Node / CRI-O
   - **Classification**: REACHABLE / PRESENT_NOT_EXPLOITABLE / PRESENT_NOT_REACHABLE / NOT_AFFECTED / UNCERTAIN
   - **Confidence**: HIGH / MEDIUM / LOW
   - **Evidence**: <source code analysis summary, call path if found>
   - **Affected versions**: 4.12.z - 4.19
   - **Assignee**: <name or "Unassigned">
   - **Tracker issues**: OCPBUGS-XXXXX, OCPBUGS-XXXXX, ...
   - **Recommended action**: <update dependency / apply patch / monitor / investigate>
   ```

2. **If `--notify-jira`**: For each CVE, post a comment on all its tracker issues:

   ```bash
   jira issue comment add OCPBUGS-XXXXX "$(cat <<'EOF'
   h3. Automated CVE Reachability Analysis

   *CVE*: CVE-XXXX-XXXXX
   *Repository*: <repo-url>
   *Classification*: REACHABLE / PRESENT_NOT_EXPLOITABLE / PRESENT_NOT_REACHABLE / NOT_AFFECTED / UNCERTAIN
   *Confidence*: HIGH / MEDIUM / LOW

   *Evidence*:
   <source code analysis summary>
   <call path if found>

   *Recommended action*: <action>

   _Generated by node-cve:triage on YYYY-MM-DD_
   EOF
   )"
   ```

   Rate limit: wait 1 second between Jira API calls to avoid throttling.

3. **If `--notify-slack`**: Post a summary to the webhook:

   Post using Slack Block Kit format (see [report-findings](../skills/report-findings/SKILL.md) Step 3 for the full payload structure).

---

### Phase 4: Summary Output

Print a final summary table to stdout:

```
Node CVE Triage - YYYY-MM-DD
═══════════════════════════════════════════════════════════════════════════
CVE              Component       Classification          Confidence  Assignee         Action
─────────────────────────────────────────────────────────────────────────────────────────────
CVE-2026-41326   Node / CRI-O    REACHABLE               HIGH        Prabhakar P.     Update dependency
CVE-2026-32281   Node / CRI-O    PRESENT_NOT_REACHABLE    HIGH        (unassigned)     Monitor
CVE-2026-35469   Node / CRI-O    UNCERTAIN                LOW         Shannon P.       Investigate manually
═══════════════════════════════════════════════════════════════════════════
Report: .work/node-cve/triage-YYYY-MM-DD/report.md
Jira comments: posted (if --notify-jira)
Slack: notified (if --notify-slack)
```

Highlight rows where:
- Classification = REACHABLE (these need priority action)
- Assignee = unassigned (these need someone to pick them up)
- Due date is within 7 days

## Arguments

- `--component <name>`: Filter to a specific OCPBUGS component. Must match a Node team component name exactly (e.g., "Node / CRI-O"). Optional.
- `--notify-jira`: Post analysis as a comment on each Jira tracker issue. Requires `JIRA_API_TOKEN` and `JIRA_USERNAME`.
- `--notify-slack`: Send a summary to the Slack webhook URL in `$SLACK_WEBHOOK`.
- `--days N`: Only include CVEs created or updated in the last N days. Default: all open CVEs.

## Examples

1. **List all open Node CVEs with reachability analysis**:
   ```text
   /node-cve:triage
   ```

2. **Triage CRI-O CVEs only and post to Jira**:
   ```text
   /node-cve:triage --component "Node / CRI-O" --notify-jira
   ```

3. **Daily headless run with Jira and Slack notification**:
   ```bash
   claude --print "/node-cve:triage --notify-jira --notify-slack"
   ```

4. **Recent CVEs only (last 7 days)**:
   ```text
   /node-cve:triage --days 7 --notify-slack
   ```

## Notes

- The Jira query uses OCPBUGS component names from `team_component_map.json` in the teams plugin.
- Each CVE typically has multiple tracker issues (one per OCP version). The command deduplicates by CVE ID and runs analysis once per unique CVE.
- Large repos like openshift/kubernetes may take longer to analyze. The command uses `--depth 1` clones for speed.
- Reachability analysis is performed by Claude reading the source code directly, not by external tools. This works across Go, Rust, and C codebases.
- Jira comments use Atlassian wiki markup (not Markdown).
- The command does not modify any code or create PRs. It only reads, analyzes, and reports.
- Reports and artifacts are saved to `.work/node-cve/` (gitignored).
