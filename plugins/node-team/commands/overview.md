---
description: Show Node team scope, responsibilities, component ownership, and plugin routing
argument-hint: "[--sub-team core|dra|kueue]"
---

## Name
node-team:overview

## Synopsis
```text
/node-team:overview [--sub-team core|dra|kueue]
```

## Description

Displays a comprehensive overview of the OpenShift Node team: what the team
owns, its responsibilities, component and repo mappings, sub-team structure,
ceremonies, upstream community involvement, active sprints, and which
specialized plugins handle which domain.

This is the entry point for understanding the Node team's scope and navigating
to the right tool for a given task.

## Implementation

1. **Read shared references:**
   - Component list and repo mappings from
     [shared/components.md](../skills/node/references/shared/components.md)
   - Sub-team assignments from the sub-teams table in `shared/components.md`
   - Sprint and board info from
     [jira.md](../skills/node/references/jira.md)
   - OCP-to-K8s version mapping from
     [shared/version-map.md](../skills/node/references/shared/version-map.md)

2. **If `--sub-team` is specified**, filter to that sub-team's components.

3. **Present a structured summary with these sections:**

### Team Mission

The Node team owns the node-level runtime stack in OpenShift: everything
between the kubelet and the container. This includes CRI-O, kubelet
customizations, Machine Config Operator (MCO), node resource managers
(CPU, memory, topology, device, NUMA), Node Problem Detector, Kueue
operator, and related components.

### Responsibilities

- Maintain downstream forks of kubelet and node-related Kubernetes components
- Own CRI-O, crun, conmonrs as the container runtime stack
- Manage Machine Config Operator for node configuration
- Handle resource management (CPU, memory, topology, device, NUMA managers)
- Triage and fix bugs in Node components (OCPBUGS)
- Track and remediate CVEs affecting Node components
- Package and ship RPMs (cri-tools, CRI-O) for each OCP release
- Participate in upstream SIG-Node community
- Onboard new team members

### Component Ownership

Table of components from `shared/components.md`:
- Component name, downstream fork, upstream repo, sub-team assignment

### Sub-teams

| Sub-team | Focus | Sprint filter |
|----------|-------|---------------|
| Core | Kubelet, CRI-O, MCO, resource managers, NPD | `Node Core` |
| DRA/Devices | Dynamic Resource Allocation, Instaslice | `Node Devices` |
| Kueue | Kueue operator, job scheduling | `OCP Kueue` |

The sub-teams are also referred to as "Green" and "Blue" in some internal
docs and calendar invites.

If roster files exist at `~/.node-assistant/team-roster-{core,dra}.json`,
include member count per sub-team.

### Ceremonies

**Daily:**
- Review Node bugs
- Review PRs
- Standups (per sub-team)
- Feature work

**Weekly:**
- 1:1s with manager (weekly or every other week)

**Sprintly (every 3 weeks):**
- Node Team Planning
- Backlog Refinement
- Sprint Retrospective
- Sprint Demos

Sprints are three weeks long and follow the OpenShift Release Dates
spreadsheet. The "AOS Main Calendar" also has the sprints scheduled.

**Every Release (~4 months):**
- RFE refinement
- Feature planning
- Feature Complete / Code Complete milestones
- Quarterly goals

### Upstream Communities

The team participates in Kubernetes SIG-Node:
- SIG-Node weekly meeting (Tuesdays 1 PM EST)
- SIG-Node CI subgroup (Wednesdays 1 PM EST)
- New Contributor Course: https://www.kubernetes.dev/docs/guide/

### Slack Channels

- `#team-node` (private, team members)
- `#forum-ocp-node` (public)
- `#forum-ocp-nisp` (public)
- `#4-dev-triage` (general questions)
- User groups: `@node-team`, `@node-green-team`, `@node-blue-team`

### Mailing List and Groups

- Mailing list: `aos-node@redhat.com`
- LDAP/Rover groups: `openshift-node-team`, `openshift-dev-node-team`
- Google group: `aos-node@redhat.com`

### Customer Support Tools

When investigating customer issues, the team uses:
- **SupportShell**: remote SSH environment for accessing customer
  must-gather archives without downloading them locally
  (`supportshell-1.sush-001.prod.us-west-2.aws.redhat.com`)
- **yank**: downloads case attachments from S3 to SupportShell
- **omc**: provides `oc`/`kubectl`-style access to must-gather data offline

For detailed setup instructions, see `/node-onboarding:checklist`.

### Active Sprints

Query the Jira Agile API (board 11478) for active sprints per the
jira.md reference. List sprint names and dates.

### Version Mapping

Show the current OCP-to-K8s version mapping from `shared/version-map.md`
for the latest 2-3 OCP releases.

### Plugin Routing

| Plugin | Command | When to use |
|--------|---------|-------------|
| `node-team` | `/node-team:overview` | Understand team scope, navigate to the right tool |
| `node-team` | `/node-team:setup` | Set up a local development environment |
| `node-team` | `/node-team:preflight` | Pre-merge checks for Node PRs |
| `node-team` | `/node-team:cleanup` | Clean up stale branches and local artifacts |
| `node-cve` | `/node-cve:triage` | CVE triage with reachability analysis |
| `node-bug` | `/node-bug:triage` | Bug triage, sub-team routing, assignment suggestions |
| `node-rpm` | `/node-rpm:bump` | Bump downstream RPM packages (cri-tools) |
| `node-onboarding` | `/node-onboarding:checklist` | New team member onboarding |

### Key Links

- Homepage: `https://source.redhat.com/groups/public/openshift_node`
- Bug dashboard: `https://redhat.atlassian.net/jira/dashboards/12991`
- Bug filter: `https://redhat.atlassian.net/issues/?filter=83963`
- Sprint board: `https://redhat.atlassian.net/jira/software/c/projects/OCPNODE/boards/11478`
- Epics board: `https://redhat.atlassian.net/jira/software/c/projects/OCPNODE/boards/4383`
- Shared Drive: `https://drive.google.com/drive/folders/1rf7-AQVRnxTWqeVrLN7TRAchEDB9Q9xP`

## Return Value

Formatted text summary of team mission, responsibilities, components,
sub-teams, ceremonies, upstream communities, active sprints, version mapping,
plugin routing, and key links.

## Examples

1. **Full team overview**:
   ```text
   /node-team:overview
   ```

2. **DRA/Devices sub-team only**:
   ```text
   /node-team:overview --sub-team dra
   ```

## Arguments

- `--sub-team <name>`: Filter to a specific sub-team (`core`, `dra`, or
  `kueue`). Optional; shows all components when omitted.
