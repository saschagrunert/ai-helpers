# Node CVE Plugin

Daily CVE triage for OpenShift Node team components. Queries open vulnerability issues from OCPBUGS, runs reachability analysis against affected repositories, and reports findings to Jira and Slack.

## Command

### `/node-cve:triage [--component <name>] [--notify-jira] [--notify-slack] [--days N]`

Triage all open CVEs for Node team components with automated reachability analysis.

**Example:**
```text
/node-cve:triage --notify-jira --notify-slack
```

**What it does:**

1. Queries OCPBUGS for open Vulnerability issues across all Node team components (CRI-O, Kubelet, MCO, etc.)
2. Deduplicates by CVE ID (each CVE has multiple version trackers)
3. Clones affected repositories and analyzes source code for reachability
4. Classifies each CVE: REACHABLE, PRESENT_NOT_EXPLOITABLE, PRESENT_NOT_REACHABLE, NOT_AFFECTED, or UNCERTAIN
5. Generates a triage report with confidence levels and recommended actions
6. Optionally posts analysis comments to all Jira tracker issues
7. Optionally sends a summary to Slack

**Arguments:**
- `--component <name>`: Filter to a specific component (e.g., "Node / CRI-O")
- `--notify-jira`: Post analysis results as comments on Jira tracker issues
- `--notify-slack`: Send summary to Slack webhook
- `--days N`: Only include CVEs updated in the last N days (default: all open)

**Output:**
- Summary table printed to stdout
- Full report at `.work/node-cve/triage-YYYY-MM-DD/report.md`
- Structured data at `.work/node-cve/triage-YYYY-MM-DD/cves.json`
- Per-CVE analysis files in `.work/node-cve/triage-YYYY-MM-DD/`

## Prerequisites

```bash
# Jira CLI
# See https://github.com/ankitpokhrel/jira-cli

# git (for cloning repos)
# curl (for --notify-slack)
```

**Environment variables:**
- `JIRA_API_TOKEN` - Jira API token (required)
- `JIRA_USERNAME` - Jira username/email (required)
- `SLACK_WEBHOOK` - Slack incoming webhook URL (required for `--notify-slack`)

## Headless Execution

Run as a scheduled job using the ai-helpers container:

```bash
podman run -it \
  -e CLAUDE_CODE_USE_VERTEX=1 \
  -e ANTHROPIC_VERTEX_PROJECT_ID=your-project \
  -e JIRA_API_TOKEN=... \
  -e JIRA_USERNAME=... \
  -e SLACK_WEBHOOK=... \
  -v ~/.config/gcloud:/home/claude/.config/gcloud:ro \
  ai-helpers --print "/node-cve:triage --notify-jira --notify-slack"
```

### OpenShift CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: node-cve-triage
  namespace: node-team
spec:
  schedule: "3 8 * * 1-5"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: triage
            image: ai-helpers:latest
            args: ["--print", "/node-cve:triage --notify-jira --notify-slack"]
            envFrom:
            - secretRef:
                name: cve-triage-secrets
          restartPolicy: OnFailure
```

## Node Team Components

The plugin covers all OCPBUGS components owned by the Node team:

| Component | Repository | Language |
|-----------|-----------|----------|
| Node / CRI-O | cri-o/cri-o | Go |
| Node / Kubelet | openshift/kubernetes | Go |
| Node / CPU manager | openshift/kubernetes | Go |
| Node / Device Manage | openshift/kubernetes | Go |
| Node / Memory manager | openshift/kubernetes | Go |
| Node / Numa aware Scheduling | openshift/kubernetes | Go |
| Node / Pod resource API | openshift/kubernetes | Go |
| Node / Topology manager | openshift/kubernetes | Go |
| Driver Toolkit | openshift/driver-toolkit | Go |
| Machine Config Operator | openshift/machine-config-operator | Go |

Additional repos detected via `pscomponent:` labels: conmon (C), conmon-rs (Rust + Go), cri-tools (Go).

## Reachability Classification

| Classification | Meaning |
|---------------|---------|
| REACHABLE | Vulnerable code path is reachable from entry points with attacker-controlled input |
| PRESENT_NOT_EXPLOITABLE | Vulnerable function is called, but only with trusted/internal data |
| PRESENT_NOT_REACHABLE | Vulnerable package is a dependency but the specific vulnerable functions are not called |
| NOT_AFFECTED | Vulnerable package is not in the dependency tree |
| UNCERTAIN | Analysis could not determine (repo too large, CVE details insufficient, etc.) |

Each classification includes a confidence level (HIGH/MEDIUM/LOW) based on the depth of source code analysis performed.
