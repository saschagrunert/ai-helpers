# Node Bug Plugin

Node-specific bug triage for OpenShift Node team components. Queries open bugs from OCPBUGS, classifies by severity and sub-team, suggests assignments based on workload, and generates triage summaries. Complements `/jira:grooming` with Node-specific JQL filters, sub-team routing, and team roster integration.

Part of the [node-team plugin family](../node-team/).

## Installation

```bash
/plugin install node-bug@ai-helpers
```

Requires the `node-team` plugin (installed automatically as a dependency).

## Command

### `/node-bug:triage [--sub-team core|devices|kueue] [--sprint <name>] [--unassigned-only]`

Query open Node bugs, classify by severity and sub-team, suggest assignments, and generate a triage summary.

**Examples:**

```text
# Full triage across all sub-teams
/node-bug:triage

# Core sub-team only, current sprint
/node-bug:triage --sub-team core --sprint "OCP Node Core Sprint 42"

# Show only unassigned bugs for DRA/Devices
/node-bug:triage --sub-team devices --unassigned-only
```

**Arguments:**

- `--sub-team core|devices|kueue`: Filter to one sub-team's components (Core, DRA/Devices, or Kueue)
- `--sprint <name>`: Filter to bugs in a specific sprint
- `--unassigned-only`: Show only untriaged or unassigned bugs

## Prerequisites

- `jira` CLI ([ankitpokhrel/jira-cli](https://github.com/ankitpokhrel/jira-cli)) configured with Jira credentials
- `JIRA_API_TOKEN` environment variable
- Optional: `~/.node-assistant/team-roster-{core,dra}.json` (for assignment suggestions)
