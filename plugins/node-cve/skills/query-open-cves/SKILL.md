---
name: query-open-cves
description: Query and deduplicate open CVE vulnerability issues from OCPBUGS for Node team components
---

## When to Use

Use this skill when Phase 1 of the `node-cve:triage` command needs to fetch all open CVE trackers from Jira and deduplicate them into a list of unique CVEs.

## Prerequisites

- `jira` CLI configured with valid credentials
- Environment variables: `JIRA_API_TOKEN`, `JIRA_USERNAME`
- Network access to Jira instance

## Implementation Steps

### Step 1: Load Node team components

The Node team components are defined in the teams plugin's `team_component_map.json`. The full list:

- Node / CRI-O
- Node / Kubelet
- Node / CPU manager
- Node / Device Manage
- Node / Memory manager
- Node / Numa aware Scheduling
- Node / Pod resource API
- Node / Topology manager
- Driver Toolkit
- Machine Config Operator

If `--component` was specified, use only that component instead of the full list.

### Step 2: Query Jira

Build and execute the JQL query:

```bash
jira issue list -q "project = OCPBUGS AND type = Vulnerability AND component in (\"Node / CRI-O\", \"Node / Kubelet\", \"Node / CPU manager\", \"Node / Device Manage\", \"Node / Memory manager\", \"Node / Numa aware Scheduling\", \"Node / Pod resource API\", \"Node / Topology manager\", \"Driver Toolkit\", \"Machine Config Operator\") AND status not in (Closed, Done, Verified)" --plain --no-headers --columns KEY,SUMMARY,STATUS,ASSIGNEE,LABELS
```

If `--days N` was specified, add `AND updated >= -${N}d` to the JQL.

Handle pagination: the `jira` CLI returns up to 100 results by default. If the result count equals 100, paginate by re-running with `--paginate 100:100`, `--paginate 200:100`, etc. until fewer than 100 results are returned.

### Step 3: Parse results

For each row in the output:
1. Extract the issue key (e.g., `OCPBUGS-85948`)
2. Extract the CVE ID from the summary using regex: `CVE-[0-9]{4}-[0-9]+`
3. Extract the OCP version from the summary brackets: `\[openshift-([^\]]+)\]`
4. Extract status and assignee

### Step 4: Deduplicate by CVE ID

Group all tracker issues by CVE ID. For each unique CVE, build a record:

```json
{
  "cve_id": "CVE-2026-32281",
  "summary": "Go crypto/x509: Denial of Service via inefficient certificate chain validation",
  "component": "Node / CRI-O",
  "status": "New",
  "assignee": "ocp-sustaining-blocked-trackers",
  "tracker_keys": ["OCPBUGS-85948", "OCPBUGS-85932", "OCPBUGS-85914", "..."],
  "affected_versions": ["4.12.z", "4.13.z", "4.14.z", "4.15.z", "4.16.z", "4.17", "4.18", "4.19"],
  "labels": ["pscomponent:cri-o", "SecurityTracking"]
}
```

Use the highest version tracker for the "primary" assignee and status (issues on newer versions are typically more actively managed).

### Step 5: Identify unassigned and urgent CVEs

Flag CVEs where:
- Assignee is a bot account (e.g., "ocp-sustaining-blocked-trackers") or empty
- Status is "New" (not yet picked up)

If due dates are available in the query, flag CVEs due within 7 days.

## Return Value

```json
{
  "skill": "query-open-cves",
  "status": "success",
  "total_trackers": 45,
  "unique_cves": 6,
  "cves": [
    {
      "cve_id": "CVE-2026-32281",
      "summary": "...",
      "component": "Node / CRI-O",
      "status": "New",
      "assignee": "ocp-sustaining-blocked-trackers",
      "is_unassigned": true,
      "tracker_keys": ["OCPBUGS-85948", "..."],
      "affected_versions": ["4.12.z", "..."],
      "labels": ["pscomponent:cri-o"]
    }
  ]
}
```

## Error Handling

- If `jira` CLI returns an error, print the error and exit. Common causes: expired API token, network issues, invalid JQL.
- If the query returns 0 results, return an empty list (not an error).
- If a CVE ID cannot be extracted from a summary, log a warning and skip that tracker.
