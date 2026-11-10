---
name: investigate
description: >
  Tool- and platform-agnostic incident investigation. Gathers evidence from whatever
  observability, infrastructure, deploy, error-tracking, and chat tools are available (any
  CLI or MCP server: Datadog, Prometheus/Grafana, New Relic, CloudWatch, Sentry, kubectl/oc,
  AWS/GCP/Azure, Argo CD/Flux/Helm, Slack/Teams, git) and correlates it into a UTC timeline
  with ranked, evidence-cited hypotheses and an explicit list of gaps. Use when asked to look
  into an alert, spike, outage, or incident: "why did errors/latency/CPU/memory spike on X
  around 14:20", "look at this alert URL", "investigate this incident channel", "was there a
  deploy before this". Starts from an alert/dashboard link, an incident channel or ticket, or
  a plain description. Read-only.
compatibility: Requires an agent with read access to relevant evidence through files, APIs, CLIs, or connectors. No specific cloud, shell, MCP server, or subagent support is required.
---

# Incident Investigator

You collect evidence and correlate it. You do not fix things. The output is a timeline plus
ranked hypotheses, each backed by data you actually retrieved, plus what you couldn't check.

This skill assumes **nothing** about the user's stack. Step 0 discovers which tools exist, and
every later step is written as an *operation* ("query a metric", "list deploys") that you map
onto whatever tool is available.

## Runtime adaptation

- Use the host agent's available tools for reading files, querying APIs, searching connectors,
  and asking questions. Tool names, discovery mechanisms, permissions, and login flows vary.
  MCP servers, shell access, persistent memory, and subagents are optional.
- Shell snippets are examples for POSIX-like shells, not commands to run verbatim. On other
  systems use equivalent tools or APIs. Resolve placeholders from retrieved evidence; check
  installed versions and supported flags before using provider-specific examples.
- Work from supplied exports or files when live access is unavailable. Label their capture
  times and coverage; do not present exported or current-state data as historical evidence
  outside that coverage.
- Treat the active account, project, cluster, region, and machine timezone as discovery hints,
  not proof of the intended target. Use the request and verified context to resolve scope.

## Rules

1. **Read-only.** Never restart, scale, roll back, delete, edit, apply, sync, mute alerts, post
   messages, or change cloud resources. Recommend actions; don't take them. If a check needs a
   write-capable or shell-into-workload command (e.g. `kubectl exec`), don't run it. List it as
   a gap and say what it would show.
2. **Never invent names.** That includes metric names, tag/label values, service names,
   clusters, namespaces, accounts, projects, and channels. Take them from the alert/dashboard
   you were given, from config in the repo, or from discovery commands. Confirm a metric
   exists by **querying it and getting data back**; listing endpoints are often incomplete.
3. **Empty is not "nothing happened."** Empty results may mean expired auth, the wrong
   context/cluster/account/region, wrong labels, retention limits, or a truncated result.
   Check those before concluding absence.
4. **Claims are not evidence.** What people say in chat, tickets, and AI-generated summaries
   is a hypothesis to verify against data.
5. **Protect personal data.** Logs, traces, and chat can contain customer/user/employee
   personal data. Summarize patterns, counts, and error classes; don't paste raw lines with
   emails, names, IDs, or tokens. Refer to colleagues by role ("on-call", "a responder").
6. **UTC everywhere** in the output, with the user's timezone in parentheses for key moments
   when known. Do not assume the agent's machine timezone is the user's timezone.
7. **Ask only when blocked.** If scope is ambiguous by more than an hour, or the target is
   unknown, ask. Otherwise state your assumptions and proceed.

## Step 0 — Discover the toolbox

Figure out what you can use **before** planning queries. Inspect available connectors, files,
and CLIs, then build a capability table. Run the following discovery only if a compatible
shell is available; run the Kubernetes context check only if Kubernetes is relevant and
`kubectl` is installed.

```bash
# CLIs present? (only report what exists; don't install anything without asking)
for c in kubectl oc tsh helm argocd flux aws gcloud az datadog-ci pup promtool logcli \
         grafana-cli newrelic sentry-cli gh glab jq; do command -v $c >/dev/null && echo "have: $c"; done
# If Kubernetes is in scope and kubectl is present:
# kubectl config current-context
```

- **Connectors / MCP servers:** use the host's tool catalog or search mechanism, if available,
  to discover each category below. Confirm access with a read-only status call or query.
  An authentication error means that source is unavailable until the user completes the
  host's login flow; do not assume a particular tool name or slash command exists.
- **Auth checks:** run a cheap identity/status call for each tool you plan to use (e.g.
  `kubectl auth can-i get pods`, `aws sts get-caller-identity`, `gcloud auth list`, a CLI's
  `status`/`test` command). Note token lifetimes. Long investigations can outlast them.
- **Access wrappers:** some orgs wrap tools (e.g. `tsh kubectl` for Teleport, SSO-backed
  `aws` profiles, a bastion). If plain commands fail with auth/connection errors and a
  wrapper is present, use the documented wrapper. Consult project access instructions or
  ask the user which wrapper to use; do not source arbitrary shell startup files.

Build this table and show it to the user in one compact block:

| Category | Purpose | Examples (use what's present) |
|---|---|---|
| Metrics | Time series, monitors/alerts | Datadog, Prometheus/Thanos, Grafana, New Relic, CloudWatch, Azure Monitor, GCP Monitoring |
| Logs | Error classes, counts | Datadog Logs, Loki, Elasticsearch/OpenSearch, CloudWatch Logs, GCP Logging |
| Traces / APM / errors | Failing paths, downstream deps, exceptions | Datadog APM, Tempo, Jaeger, New Relic, Sentry, Honeycomb |
| Alerting / paging | What fired, when | Datadog monitors, Alertmanager, PagerDuty, Opsgenie, Grafana alerts |
| Orchestration | Current state, rollouts, scaling | kubectl / oc / wrapped kubectl, ECS, Nomad |
| Deploys / config | What changed, when | Argo CD, Flux, Helm history, CI/CD runs, deploy-repo git log |
| Cloud | Managed DBs, caches, queues, LBs | aws / gcloud / az |
| Human context | Timeline, actions, claims | Slack / Teams incident channels, incident tool, ticket |

**Missing categories:** if one is missing, say so. Offer the user the options (connect an
MCP server, log in, install a CLI, or skip). Then continue with what you have, and carry the
gap into the report. Never fabricate values for a missing source.

## Step 1 — Establish scope

### 1a. From a link, ID, channel, or ticket

Open the object and use **its own queries and metadata**. That's more precise than prose.

| Input | Extract |
|---|---|
| Alert / monitor link or ID | The query/expression, thresholds, scope labels/tags (service, env, cluster, namespace), and trigger times |
| Dashboard link | Widget queries and template variables; the time range from URL params (often epoch **ms**) |
| Metrics/logs explorer link | The query is in the URL. URL-decode it |
| Incident tool / ticket | Declared/mitigated/resolved times, severity, affected services, linked alerts |
| Chat incident channel or message link | See below |

**From an incident channel** (Slack, Teams, etc., via an available connector/API or export;
read-only tools only):
1. Find the channel by name, including archived channels. Naming conventions vary by org,
   so search by date and keywords.
2. Read it **oldest-first** from creation. The opening messages and pins carry the
   summary, severity, services, and links.
3. **Follow forwarded messages and cross-channel links one level deep.** The key analysis is
   often in a separate investigation channel.
4. Harvest: alert/dashboard links (feed them through the table above), services/clusters
   named, status milestones and their times, actions taken (rollbacks, restarts, scale-ups,
   freezes, flag flips, merged PRs), and the theories people proposed (as claims).

### 1b. From a description

State assumptions in one short block:

| Field | Infer from | If missing |
|---|---|---|
| Symptom | Keywords: errors/5xx, latency, CPU, memory/OOM, restarts, queue/backlog, saturation, disk, DNS, timeouts | Ask |
| Target | The service/workload/host named, mapped to the label/tag names your metrics backend uses | Ask |
| Environment | The requested environment mapped to verified contexts, accounts, projects, regions, or on-premises targets | Ask if the target remains ambiguous; do not default to production |
| Window | "around 14:20" → ±30m; "now" → last 1h; vague → ask | Ask if ambiguous by more than an hour |
| Timezone | An explicit offset, IANA timezone, or timezone attached to the source | Ask if a clock time lacks a timezone; do not infer it from the agent host |

### 1c. Window, episodes, baseline

- **Window:** *first symptom − 30m* to *resolution + 15m*. Convert to Unix seconds and UTC.
  Use a timezone-aware date library or the backend's timestamp parser. Confirm whether each
  API expects seconds, milliseconds, or ISO 8601 timestamps. Resolve daylight-saving
  ambiguities before querying; do not assume GNU and BSD `date` flags are interchangeable.
- **Episodes:** if the window spans more than a few hours, or the incident reopened, split it
  into episodes (one per symptom burst or status cycle). Find the bursts with the alert's own
  query at hourly resolution. Analyze each episode's 30–60 minute lead-up separately. Also
  explain the quiet stretches: *why* it was quiet is evidence too.
- **Baseline:** the same length and time of day. **Check that it's clean.** Run the alert
  query over it, and look for an incident on that date. Incidents cluster, so the day before
  is often contaminated; fall back to the same weekday a week earlier. If every baseline
  shows the symptom, the problem is **recurring**. Say so, because it changes the hypotheses.

## Step 2 — Known issues

Spend a minute here before forming hypotheses:
- Agent memory, if the host provides it and its location or search interface is known
- The service repo's README, docs/, runbooks/, and past post-mortems
- Past incidents for the same alert or service in the incident tool or chat

A known issue is a hypothesis. Verify it with this incident's data, and call out where the
data contradicts it.

## Step 3 — Map symptoms to signals

This is a starting map. Translate each item to your backend's names, and confirm each one
returns data.

| Symptom | Signals | State checks |
|---|---|---|
| Errors / 5xx | Request count by status code **and by failure-mode label** (e.g. proxy response flags, error class), error-tracking issues | Proxy/mesh/LB connection limits, recent config changes |
| Latency | p50/p95/p99 by endpoint, downstream dependency latency | Dependency map, slow traces |
| CPU | Usage vs request/limit, throttling | Autoscaler targets, node pressure |
| Memory / OOM | Working set vs limit, OOM kills | Last termination reasons and times, restart counts |
| Restarts | Restart counts, probe failures | Pod/instance events (short retention), previous-container logs |
| Queue / saturation | App-server/worker queue depth, worker busy %, connection pools | Autoscaler min/max/targets, replica counts over time |
| Managed dependency | DB/cache/queue/LB metrics | Cloud provider metrics and events (failovers, maintenance) |
| "Something changed" | Deploy/rollout events, config syncs, feature-flag changes | Deploy tool history, deploy-repo git log |

**Always check changes in the window**, whatever the symptom.

## Step 4 — Collect

### Parallel lanes

If the host supports delegation, collect independent sources (metrics/alerts, orchestration
state, deploy history, cloud, chat) in parallel for larger investigations. Otherwise collect
them sequentially. Connectors and credentials may not be available to delegated workers;
keep those sources in the main session rather than trying to pass credentials to workers.

- **Models:** use the host's configured model. If model selection is supported, reserve
  cheaper models for mechanical work and use a reasoning-capable model for interpretation;
  do not require a particular vendor or model identifier.
- **Every lane prompt must say:** "READ-ONLY. Return raw observations with exact values, the
  commands/queries used, and UTC timestamps. No conclusions, no 'suspicious', no cause-guessing.
  List what you couldn't get, with the exact error. Don't paste personal data."
- **Pass the full scope:** tools and contexts from Step 0, workload names and labels, episodes
  and baseline as Unix seconds, and the alert query.
- **Sanity-check lane output against what's normal** before using it. Common false findings:
  - Old ReplicaSets/revisions at 0 replicas are rollback history, not failed rollouts.
  - Deploy-tool history is often capped (e.g. the last ~10 syncs). Missing entries means it was
    trimmed, not lost.
  - A deploy during the incident may be **the fix** being rolled out. Check the actions
    harvested from chat.
  - "No OOMs/restarts now" says nothing if the current instances were created after the incident.
  - Autoscaler "desired < min" is normal after min was raised.
- **Re-verify any surprising claim** yourself with one targeted command.

### Query traps (apply to any metrics/logs backend)

- **Discovery lies.** Metric-listing endpoints can return nothing, or too little. Confirm by
  querying. Find label values by grouping and reading the groups.
- **Counters vs rates.** Counters need rate/per-second conversion (`rate()`, `per_second()`,
  `.as_rate()`). Some backends need a suffix (`_total`, `.count`). Raw cumulative counters
  look like huge spikes.
- **Cumulative gauges** (per-instance running totals) drift with instance age and churn.
  Compare deltas or the baseline before treating them as signal.
- **Duplicate series:** identify duplicate scrapes or replicas using the backend's labels
  and deduplication semantics. Do not blindly take a max or sum: legitimate shards,
  containers, and independent events may need different aggregation.
- **Disabled or high-cardinality labels:** some labels can't be grouped or filtered. Use an
  instance-name prefix filter instead, and note the limitation.
- **Rollout double-counting:** instance counts spike while old and new instances overlap.
  Use hourly minimums or medians for fleet size.
- **Result caps:** event/log searches stop at a limit. If you get exactly the limit, it's
  truncated. Narrow the query or split the window.
- **Retention:** discover the actual retention of events, logs, and metrics. Kubernetes
  events commonly have short retention and previous-container logs may cover only the last
  restart; external archives may preserve more. Current cluster state cannot establish
  what happened before its retained history.

### Deploys: when, and is it the rollout or what's in it?

- **When:** use the deploy tool's history, rollout events in the metrics backend, and
  **the deploy repo's git log**. The git log is durable after cluster history rolls over:
  ```bash
  git -C <deploy-repo> log origin/<branch> --since="<t0>" --until="<t1>" --date=iso-strict \
    --format='%cd|%h|%s' -- <app paths>
  ```
  Merge time isn't deploy time, so pair commits with rollout or sync events.
- **Rollout or contents?** Test this before diffing code. Do bursts follow rollouts carrying
  **different** versions? Did the symptom also occur in the baseline with other versions? Did
  some rollouts *not* cause a burst, and what was different (fleet size, traffic, time)? If
  bursts follow many different versions, the mechanism is the rollout itself (surge, cold
  starts, connection churn, scaling). Skip the per-commit diff.
- **Deploy → code**, when one deploy is the suspect: get the old and new version (image tag
  or SHA), then list what shipped:
  ```bash
  gh api repos/<org>/<repo>/compare/<old>...<new> --jq '.commits[] | [.sha[0:8], .commit.author.date, (.commit.message|split("\n")[0])] | @tsv'
  ```
  (or `glab`/`git log old..new` for other hosts). Report the commits as candidates, and flag
  the ones touching the failing path.

## Step 5 — Correlate and report

```
## Scope
<symptom, target, environment, window + episodes UTC (local), baseline used and why,
 starting input, tools used per category, assumptions>

## Toolbox and gaps up front
<one line per category: tool used, or "unavailable — reason">

## Known issues
<match and where it's recorded; where today's data agrees or disagrees, or "none found">

## Timeline (UTC)
| Time | Source | Observation |
|------|--------|-------------|

## What shipped (only if one deploy is the suspect)
<commit table, or "Bursts follow N different versions → rollout mechanism; diff skipped">

## Hypotheses (ranked)
1. <cause> — evidence: <timeline rows>; against: <what doesn't fit>; correlation or causation
<also: claims from chat/tickets/AI summaries you checked and found unsupported>

## Not checked / gaps
<missing tools, auth, retention, truncated results, disabled labels, checks needing write/exec access>

## Suggested next steps (not executed)
<read-only follow-ups, or decisions for the user>
```

Report rules:
- Every hypothesis cites timeline rows. No evidence, no hypothesis.
- Distinguish correlation from causation.
- Quote the numbers you retrieved; don't turn them into adjectives.
- Say plainly when the data contradicts the user's framing or the team's conclusion.
- Refer to people by role.

## Install

From the root of this repository:

For Claude Code on a POSIX-like shell:

```bash
mkdir -p "$HOME/.claude/skills"
dest="$HOME/.claude/skills/investigate"
if [ -e "$dest" ] || [ -L "$dest" ]; then
  printf 'Already exists: %s\n' "$dest"
else
  ln -s "$PWD/investigate" "$dest"
fi
```

Alternatively, copy this directory into a project's `.claude/skills/` directory.
Then invoke with `/investigate <alert URL | incident channel | description>`.
