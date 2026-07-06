---
description: Query open Node bugs, classify by severity and sub-team, suggest assignments, and generate a triage summary
argument-hint: "[--sub-team core|devices|kueue] [--sprint <name>] [--unassigned-only]"
---

## Name
node-bug:triage

## Synopsis
```text
/node-bug:triage [--sub-team core|devices|kueue] [--sprint "OCP Node Core Sprint 42"] [--unassigned-only]
```

## Description

Queries all open bugs in OCPBUGS for Node team components using the "Node Bugs" saved filter, classifies them into triage buckets (Release Blockers, Customer Escalations, Potential Blockers, Component Regressions, Untriaged), routes each bug to the correct sub-team, and suggests assignments based on current workload.

Designed for both interactive triage sessions and headless execution via `claude --print`.

## Implementation

### Phase 0: Setup and Argument Parsing

1. **Parse Arguments**
   - `--sub-team core|devices|kueue`: Filter results to one sub-team's components. Optional. Read sub-team component lists from the sub-teams table in [shared/components.md](../../node-team/skills/node/references/shared/components.md) rather than hardcoding names.
     - `core`: all Node components not listed under another sub-team
     - `devices`: components listed under DRA/Devices in the sub-teams table
     - `kueue`: components listed under Kueue in the sub-teams table
   - `--sprint <name>`: Filter to bugs in a specific sprint (e.g., "OCP Node Core Sprint 42"). Optional.
   - `--unassigned-only`: Show only untriaged or unassigned bugs. Optional.
2. **Validate Tools**

   ```bash
   which jira 2>/dev/null || echo "MISSING: jira CLI"
   ```

   If `jira` is missing, display installation instructions ([ankitpokhrel/jira-cli](https://github.com/ankitpokhrel/jira-cli)) and exit.

   Verify credentials:

   ```bash
   jira me
   ```

   Exit with error if `jira me` fails (not configured, expired token, or network issue).

3. **Create work directory**: `mkdir -p .work/node-bug/triage-$(date +%Y-%m-%d)`

---

### Phase 1: Query Bugs

1. **Use the Jira saved filter** "Node Bugs" (ID 83963) for the base query. The filter defines which components are in scope.

2. **Build base JQL**:

   ```text
   filter = "Node Bugs" AND status not in (Closed, Done, Verified)
   ```

   Apply optional filters (read sub-team component names from the sub-teams table in [shared/components.md](../../node-team/skills/node/references/shared/components.md)):
   - If `--sub-team devices`: append `AND component in (<DRA/Devices components>)`
   - If `--sub-team kueue`: append `AND component in (<Kueue components>)`
   - If `--sub-team core`: append `AND component not in (<DRA/Devices components>, <Kueue components>)`
   - If `--sprint <name>`: append `AND sprint = "<name>"`
   - If `--unassigned-only`: append `AND (assignee is EMPTY OR assignee in ("aos-node@redhat.com") OR priority = Undefined OR "Release Blocker" = Proposed)`

3. **Execute the query** using the `jira` CLI:

   ```bash
   jira issue list -q "<constructed JQL>" \
     --plain --no-headers --columns KEY,SUMMARY,COMPONENT,STATUS,ASSIGNEE,LABELS,PRIORITY
   ```

   Handle pagination: the `jira` CLI returns up to 100 results by default. If the result count equals 100, paginate by re-running with `--paginate 100:100`, `--paginate 200:100`, etc. until fewer than 100 results are returned.

4. **Print intermediate summary**: "Found N open bugs for Node team components."

**Decision Point:**
- IF 0 bugs found: print "No open bugs matching filters." and exit.
- IF bugs found: continue to Phase 2.

---

### Phase 2: Classify and Route

1. **Route each bug to its sub-team** using the sub-teams table from [shared/components.md](../../node-team/skills/node/references/shared/components.md):
   - DRA/Devices: bugs whose component appears in the DRA/Devices row of the sub-teams table
   - Kueue: bugs whose component appears in the Kueue row of the sub-teams table
   - Core: all remaining Node components

2. **Classify each bug into triage buckets** using JQL classification queries. Run each query with the same base JQL from Phase 1 (including any `--sub-team`, `--sprint`, or `--unassigned-only` filters) plus a bucket-specific condition. Use the [Bug Triage Definitions](../../node-team/skills/node/references/jira.md) as the source of truth.

   For each bucket, run:
   ```bash
   jira issue list -q "<base JQL> AND <bucket condition>" \
     --plain --no-headers --columns KEY
   ```

   Bucket conditions:
   - **Release Blockers**: `AND "Release Blocker" = Approved`
   - **Potential Blockers**: `AND priority = Blocker AND "Release Blocker" is EMPTY`
   - **Customer Escalations**: `AND ("SFDC Cases Counter" is not EMPTY OR "Customer Impact" = "Customer Escalated")`
   - **Component Regressions**: `AND labels = component-regression`
   - **Untriaged**: `AND (priority = Undefined OR "Release Blocker" = Proposed OR assignee in ("aos-node@redhat.com"))`
   - **Other**: bugs from the main query that do not appear in any of the above buckets

   A bug can appear in multiple buckets (e.g., a release blocker that is also a customer escalation). Count it in each applicable bucket.

   For Customer Escalation bugs, fetch the SFDC Cases Counter for display:
   ```bash
   jira issue view OCPBUGS-XXXXX --raw
   ```
   Parse `customfield_10978` from the JSON output to show "N support cases" in the report.

3. **Assignment suggestions** (when team roster files exist):

   Load team rosters from `~/.node-assistant/team-roster-{core,dra,kueue}.json`. If roster files do not exist, skip assignment suggestions and print "Roster files not found, skipping assignment suggestions."

   Query all team members' open bug counts:
   ```bash
   jira issue list -q "filter = \"Node Bugs\" AND status not in (Closed, Done, Verified) AND assignee in (<all roster members>)" \
     --plain --no-headers --columns KEY,ASSIGNEE
   ```
   Handle pagination the same way as Phase 1. Group results by assignee in code to build a workload map.

   For each unassigned or mailing-list-assigned bug:
   - Determine the correct sub-team from step 1
   - Exclude "Node Team Bot Account" from suggestions
   - Suggest the team member with the fewest open bugs from the appropriate sub-team roster

---

### Phase 3: Generate Triage Summary

1. **Print the triage summary** grouped by classification with counts:

   ```text
   Node Bug Triage (N bugs)

   Release Blockers: X
   Customer Escalations: X
   Potential Blockers: X
   Component Regressions: X
   Untriaged: X
   Other: X

   --- Release Blockers ---
   * OCPBUGS-XXXXX: <summary> (Component, Severity, Assignee)
   * OCPBUGS-XXXXX: <summary> (Component, Severity, Unassigned -> suggested: <name>)

   --- Customer Escalations ---
   * OCPBUGS-XXXXX: <summary> (Component, N support cases, Assignee)

   --- Potential Blockers ---
   * OCPBUGS-XXXXX: <summary> (Component, Severity, Assignee)

   --- Component Regressions ---
   * OCPBUGS-XXXXX: <summary> (Component, Assignee)

   --- Untriaged ---
   * OCPBUGS-XXXXX: <summary> (Component, Unassigned -> suggested: <name>)

   --- Other ---
   * OCPBUGS-XXXXX: <summary> (Component, Priority, Assignee)

   Workload Distribution:
   | Team Member | Open Bugs | Sub-team |
   |-------------|-----------|----------|
   | <name>      | N         | Core     |
   | <name>      | N         | DRA      |

   Filter: https://redhat.atlassian.net/issues/?filter=83963
   Dashboard: https://redhat.atlassian.net/jira/dashboards/12991
   ```

   Omit empty sections. Include the workload distribution table only when roster files are available. Show assignment suggestions inline for unassigned bugs.

2. **Save the report** to `.work/node-bug/triage-$(date +%Y-%m-%d)/report.md`.

## Return Value

Prints the triage summary to stdout. Saves a report file to `.work/node-bug/triage-$(date +%Y-%m-%d)/report.md`. No write operations are performed on Jira (read-only).

## Examples

1. **Full triage across all sub-teams**:
   ```text
   /node-bug:triage
   ```

2. **Core sub-team bugs in the current sprint**:
   ```text
   /node-bug:triage --sub-team core --sprint "OCP Node Core Sprint 42"
   ```

3. **Unassigned DRA/Devices bugs only**:
   ```text
   /node-bug:triage --sub-team devices --unassigned-only
   ```

4. **Headless run for CI/scheduled jobs**:
   ```bash
   claude --print "/node-bug:triage --unassigned-only"
   ```

## Arguments

- `--sub-team core|devices|kueue`: Filter to one sub-team's components. `core` includes all Node components not listed under another sub-team. `devices` includes only components listed under DRA/Devices. `kueue` includes only components listed under Kueue. Sub-team definitions are in the [sub-teams table](../../node-team/skills/node/references/shared/components.md). Optional.
- `--sprint <name>`: Filter to bugs in a specific sprint. Use the exact sprint name from Jira (e.g., "OCP Node Core Sprint 42"). Optional.
- `--unassigned-only`: Show only bugs that are untriaged or unassigned (priority Undefined, Release Blocker Proposed, assignee is the mailing list, or assignee is empty). Optional.

## Notes

- The Jira query uses the "Node Bugs" saved filter (ID 83963) as the base. This filter is maintained in Jira and defines which bugs are in scope. The "Node Bugs" dashboard (ID 12991) provides a visual overview at `https://redhat.atlassian.net/jira/dashboards/12991`.
- Sub-team routing uses the sub-teams table from the [node-team shared components reference](../../node-team/skills/node/references/shared/components.md). Core owns all Node components not listed under DRA/Devices or Kueue.
- Assignment suggestions require team roster files at `~/.node-assistant/team-roster-{core,dra,kueue}.json`. Sync these from Jira config issue OCPNODE-4230 (see [jira.md](../../node-team/skills/node/references/jira.md) Team Roster section).
- The command is read-only. It does not modify bugs, change assignments, or transition issues. All suggestions are advisory.
- Reports and artifacts are saved to `.work/node-bug/` (gitignored).
- For CVE-specific triage with reachability analysis, use `/node-cve:triage` instead.
