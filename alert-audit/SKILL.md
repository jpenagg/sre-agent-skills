---
name: alert-audit
description: >
  Audits alerting for noise and blind spots: compares each alert's definition and routing
  with how it actually behaved (how often it fired, for how long, whether it flapped, whether
  it was silenced, and whether anyone acted on it), then recommends keep, tune, demote,
  merge, fix, or delete with backtested changes, and lists symptoms and past incidents that
  no alert covers. Tool- and platform-agnostic: uses whatever is available (Datadog,
  Prometheus/Alertmanager, Grafana, CloudWatch, New Relic, Sentry, PagerDuty/Opsgenie/
  incident.io, Slack/Teams, rule files and IaC). Use for "which alerts are noisy", "what's
  paging us most", "reduce on-call load", "is this alert's threshold right", "which alerts
  never fire", "do we have alerting gaps on X". Read-only.
compatibility: Requires an agent with read access to alert definitions and their firing history through files, APIs, CLIs, or connectors. No specific monitoring backend, paging tool, shell, MCP server, or subagent support is required.
---

# Alert Audit

You measure how each alert behaved, compare that to what it is for, and recommend what to do
with it. You do not edit, mute, or delete anything. The output is a per-alert table of
verdicts and proposed changes, each backed by firing and response history you retrieved,
plus the gaps in coverage and what you couldn't check.

This skill assumes **nothing** about the user's stack. Step 0 discovers which tools exist;
later steps are written as *operations* ("list firing episodes", "resolve the route") that
you map onto whatever tool is available. Prometheus and AWS examples are given because
they're common. Use the equivalent for other backends.

## Runtime adaptation

- Use the host agent's available file, API, connector-discovery, and question tools. No
  particular tool name, MCP server, model vendor, or delegation feature is required.
- Shell and query snippets illustrate backend-specific operations. Use equivalent native
  commands or APIs elsewhere. Resolve placeholders from discovery; verify command versions,
  supported flags, and query semantics before use.
- Supplied exports (monitor definitions, a paging-tool incident export, an alert-channel
  export, rule files) can substitute for live tools. Record their capture time, lookback, and
  coverage, and limit conclusions to that evidence.
- An audit of definitions alone, without firing history, is a lint, not an audit. Say which
  one you delivered.

## Rules

1. **Read-only.** Never edit, mute, silence, delete, or re-route an alert, acknowledge or
   resolve a page, or change an escalation policy or schedule. Never edit rule files or IaC.
   Recommend changes and point to where they'd be made. If the user later asks for a PR,
   that's a separate, explicit request.
2. **Never invent names or numbers.** Alert IDs, metric names, labels, routes, and teams come
   from discovery, rule files, or the user. Firing counts, durations, and response rates come
   from history you retrieved.
3. **Quiet is not healthy, and quiet is not useless.** An alert with no fires may be the most
   important one you have (disk full, certificate expiry, backup failure, a dead-man's
   switch), or it may be unable to fire at all. Never recommend deletion for silence alone;
   check that it *can* fire (Step 2).
4. **Noisy is not always a bad alert.** Frequent fires may be an honest report of a system
   that keeps breaking. Separate "the alert is wrong" from "the alert is right and the system
   needs fixing" before proposing a threshold change.
5. **Empty is not "never fired."** No history may mean limited retention, state history not
   enabled, an alert recreated under a new ID, a muted period, or a query against the wrong
   account/region/datasource. Check those before concluding absence.
6. **Enough history.** Default lookback is **30 days** (minimum 14), so it covers weekly
   cycles and on-call rotations. Use up to 90 days for "never fires" verdicts when retention
   allows. Less than 14 days of history → verdict "insufficient data".
7. **Actionability needs response evidence.** Whether an alert was acted on comes from the
   paging tool, incident tool, or chat. Without it, write "response unknown"; do not call an
   alert unactionable from firing counts alone. An acknowledgement is not an action.
8. **Test proposed changes.** A new threshold or window is backtested against the lookback
   (Step 4) or labeled "untested". Never propose a number from memory or habit.
9. **Audit alerts, not people.** Report response by rotation or role, never by individual,
   and don't rank responders by acknowledgement time. Don't paste raw alert payloads that
   contain personal data, customer identifiers, or tokens.
10. **Ask only when blocked.** Use sensible defaults (below), state them, and proceed.

## Step 0 — Discover the toolbox

```bash
# Optional discovery for a compatible shell; API connectors and offline files also work.
for c in promtool amtool logcli pup datadog-ci aws gcloud az kubectl oc tsh newrelic \
         sentry-cli pd terraform gh glab jq; do
  command -v "$c" >/dev/null && echo "HAVE $c"
done
```

- **Connectors / MCP servers:** inspect the host's tool catalog or discovery interface for
  monitoring, paging, incident, and chat sources. Confirm access with a read-only status call
  or query. Use the host's documented login flow if authentication is required.
- **Auth and scope:** confirm identity and target before querying (`aws sts
  get-caller-identity`, `kubectl config current-context`, the monitoring org or datasource
  you are pointed at). Access wrappers (e.g. `tsh kubectl`, `aws-vault exec`) replace the
  plain command; use whichever the environment requires.
- **Rule files:** check the current repo and the user's code directory for alert definitions
  (Terraform monitor resources, `PrometheusRule` manifests, Alertmanager config, Grafana
  provisioning, mixins or vendored chart rules). This tells you where a change would be made
  and whether an alert is hand-written or generated.

Map what you found to capabilities:

| Capability | Used for | Examples (use what's present) |
|---|---|---|
| Alert definitions | Query, threshold, window, no-data behavior, labels | Datadog monitors, Prometheus rules (`/api/v1/rules`), Grafana alert rules, CloudWatch alarms, New Relic conditions, Sentry alert rules, rule files |
| Firing history | Episodes, durations, flapping | Monitor state history or events, Prometheus `ALERTS` series, Grafana state history, `aws cloudwatch describe-alarm-history` |
| Routing | Who or what is actually notified | Alertmanager routes and receivers, notification handles in monitor messages, alarm actions, paging-tool services and escalation policies |
| Response history | Acknowledged, escalated, resolved by whom (role), linked incidents | PagerDuty, Opsgenie, incident.io, alert channels in Slack / Teams |
| Silences and mutes | Alerts that are switched off in practice | Alertmanager silences, Datadog downtimes, paging-tool maintenance windows, disabled alarm actions |
| Underlying metrics | Backtesting proposed thresholds | The metrics backend the alerts query |
| Services, SLOs, incidents | Coverage gaps | Service catalog, workload inventory, SLO definitions, incident history, postmortems |

Show the user a one-line summary of what's available and what's missing. Say what each gap
costs: no firing history means definitions-only lint; no response history means noise can be
measured but actionability cannot; no metrics access means proposals are untested.

## Step 1 — Scope and inventory

### 1a. What's in scope

From the prompt and verified context, resolve the team, service, environment, monitor tag,
route, or paging service to audit, and the alert sources that cover it. Do not default to
every alert in the organization, to production, or to the datasource the CLI happens to
point at. Ask if the target is ambiguous; otherwise state the scope and coverage limits
before querying.

If the user named one alert ("is the checkout latency alert too noisy"), resolve it to its ID
and skip broad inventory.

### 1b. Inventory

For each alert, record: ID and name, the query or expression, threshold, evaluation window or
`for:` duration, no-data and recovery behavior, declared severity, labels/tags (service,
team, env), runbook link, owner, created and last-modified times, where it is defined (rule
file path, or "managed in the UI"), and whether it is muted now. Examples:

```bash
# Prometheus / Alertmanager
curl -s "$PROM_URL/api/v1/rules?type=alert" | jq '.data.groups[].rules[] | {name, query, duration, labels, health}'
amtool --alertmanager.url="$AM_URL" config routes show
amtool --alertmanager.url="$AM_URL" silence query
# AWS CloudWatch
aws cloudwatch describe-alarms \
  --query 'MetricAlarms[].{name:AlarmName,state:StateValue,enabled:ActionsEnabled,alarm:AlarmActions,ok:OKActions,missing:TreatMissingData,periods:EvaluationPeriods,threshold:Threshold}'
```

**Resolve the route to what actually happens.** Declared severity is a label; the route is
the behavior. For each alert, determine whether a fire **pages a person**, **opens a
ticket**, **posts to a channel**, **triggers automation**, or **does nothing**.

- Alertmanager: test the alert's labels against the routing tree
  (`amtool config routes test <label>=<value> ...`) and read the receiver's integrations.
- Monitor-based tools: read the notification handles in the message and follow them to the
  paging service, its escalation policy, and whether anyone is on the schedule.
- CloudWatch: read the alarm's actions. An alarm with actions disabled, or with no alarm
  action, notifies no one.
- Check the target still exists where you can: archived channels, disabled paging services,
  and deleted integration keys all make an alert silent.

**Group, then rank.** With large estates, don't analyze alphabetically. Rank by pages
generated (paging-tool history) or by firing episodes, and analyze the top items first: a
small number of alerts usually produce most of the load. Say how many you skipped.

## Step 2 — Measure behavior

Get, per alert over the lookback:

| Measure | Definition | Why |
|---|---|---|
| Episodes | Transitions into firing | Raw load |
| Firing time | Total time in firing state; median and p90 episode duration | Short episodes are rarely actionable; a permanent one is ignored |
| Self-resolved share | Episodes that cleared without a recorded human action | The core noise signal |
| Flapping | Episodes that re-fire shortly after resolving; peak episodes per hour | Missing hysteresis or a threshold on the noise floor |
| Timing pattern | Distribution by hour and weekday; overlap with deploys, batch jobs, backups, maintenance | Points to the cause of the noise |
| Per-group spread | For multi-group alerts, episodes per host / pod / tenant / region | One bad member can produce most fires |
| Co-firing | Other alerts that start within a few minutes of this one, repeatedly | Duplicates and cause/symptom pairs |
| Notifications | Pages, tickets, or messages actually sent; pages outside working hours | Grouping and repeat intervals make fires ≠ notifications |
| Response | Acknowledged, time to acknowledge, escalated, resolved by a person or automatically, linked to an incident, followed by an action | Whether the alert leads to anything |
| Muted time | Share of the lookback under a silence, downtime, or disabled state | An alert muted most of the time is already deleted |
| Able to fire | Does the query return data now? Is the threshold reachable given observed values? | An alert on a renamed metric can never fire |
| Real incidents | Incidents in the lookback, whether this alert fired for each, and when relative to impact start | True positives, and what a change must not lose |

Prometheus keeps alert state as a sampled series. These are candidates; verify the result
against one alert whose recent history you know:

```promql
# Firing time per alert: samples × the rule group's evaluation interval
sum by (alertname) (count_over_time(ALERTS{alertstate="firing"}[30d]))
# Episodes per alert: firing now but not one step earlier (set the step to the evaluation interval)
sum by (alertname) (sum_over_time((ALERTS{alertstate="firing"} unless ALERTS{alertstate="firing"} offset 1m)[30d:1m]))
```

```bash
aws cloudwatch describe-alarm-history --alarm-name <name> --history-item-type StateUpdate \
  --start-date <t0> --end-date <t1> --query 'AlarmHistoryItems[].{t:Timestamp,s:HistorySummary}'
```

**Evidence that an alert was acted on**, weakest to strongest: acknowledged; a reply or
thread in the alert channel; escalated or reassigned; linked to an incident; followed within
the episode by a change that plausibly addresses it (deploy, rollback, scale, failover,
ticket). Report the strongest level found, and "response unknown" when you have no source.

### Measurement traps

- **Sampled state misses short fires.** A state series evaluated every minute cannot show a
  30-second fire. Prefer the backend's state-transition history when it exists.
- **History has its own retention**, often shorter than metrics. Discover it and confirm it
  covers the lookback. CloudWatch alarm history and Kubernetes-stored state are both limited.
- **Recreated alerts lose their history.** IaC that replaces rather than updates a monitor
  resets its ID. Check the created time; an alert younger than the lookback has a shorter
  effective window.
- **A recent edit splits the history.** If the threshold or window changed during the
  lookback (modified time, rule-file git log), judge only the period since the change.
- **Fires are not notifications.** Alertmanager grouping, inhibition, and `repeat_interval`,
  or a monitor's renotify setting, change how many pages one episode produces. A long-firing
  alert with a short repeat interval pages many times.
- **Paging tools merge and auto-resolve.** Deduplication keys fold distinct alerts into one
  incident; map incidents back to alerts through the payload or key, not by title. Auto-
  resolve and acknowledgement timeouts make "resolved" look like someone handled it.
- **Acknowledged is not acted on.** An acknowledgement followed by an automatic resolve a
  few minutes later is a self-resolved episode.
- **Severity labels drift from routes.** "Critical" alerts that post to a channel and
  "warning" alerts that page are both common. Use the resolved route from Step 1.
- **Working hours need a timezone.** Use the rotation's timezone if you can retrieve it.
  Otherwise report by UTC hour and say that off-hours counts are not computed.

## Step 3 — Classify

Default conditions (state them, and let the user override). An alert can carry more than
one; report the one that most needs action first.

| Verdict | Condition (over the lookback) |
|---|---|
| **Insufficient data** | < 14 days of usable history, or firing history unavailable |
| **Broken** | Cannot fire (query returns no data, metric or label no longer exists, threshold outside any observed value by construction), or fires to nobody (no receiver, dead target, actions disabled) |
| **Silenced** | Muted for more than half the lookback, or by a silence with no expiry or owner |
| **Stuck** | Firing continuously for more than 7 days |
| **Flapping** | 3 or more episodes within one hour, on 3 or more separate days |
| **Noisy** | 10 or more episodes, and at least half self-resolved within 10 minutes |
| **Unactionable** | 5 or more episodes with response history available, and none shows more than an acknowledgement |
| **Duplicate** | At least 80% of its episodes start within 5 minutes of another alert's, and both notify the same people |
| **Misrouted** | Pages, but episodes are consistently handled the next working day or reassigned to another team; or does not page, but was the earliest signal of a declared incident |
| **Quiet — verified** | No fires, and you confirmed it can fire and its route works |
| **Healthy** | Fires rarely, episodes map to actions or incidents, routed to the people who act, runbook present |

These are starting heuristics, not policy. A team with one page a month and a team with
forty a week need different bars. Broken, Silenced, and Stuck alerts are **detection gaps**:
rank them above noise.

Then identify the **pattern** behind each Noisy, Flapping, Unactionable, or Stuck alert. The
pattern, not the count, decides the fix:

| Pattern in the history | Likely cause | Lever |
|---|---|---|
| Short spikes that clear alone | Threshold inside normal peaks; evaluation window too short | Longer window or `for:`, threshold above the observed normal peak |
| Oscillates around the threshold | No hysteresis | Separate recovery threshold; longer window |
| Same hour daily or weekly | Batch job, backup, cron, traffic trough | Fix or reschedule the job; guard the condition; scheduled downtime as a last resort |
| Fires during rollouts | Restarts, cold starts, old and new instances overlapping | Rollout-tolerant condition; fix readiness or surge settings |
| Ratio alert at low traffic | A handful of requests swing an error rate | Minimum-volume guard on the denominator |
| One member produces most fires | A bad host, pod, tenant, or region | Fix that member; don't raise the global threshold |
| Always fires with another alert | Cause and symptom both page | Group or inhibit; page on the symptom, demote the cause |
| Cause metric with no user impact | CPU, memory, or queue depth paging while user-facing signals stay normal | Demote to ticket or dashboard; page on the user-facing symptom or SLO burn |
| Long-running, nobody responds | Known issue, accepted risk, or an abandoned system | Fix it, lower it to a ticket with an owner, or retire it with the system |
| Fires, someone acts, it recurs | The system has a recurring fault | Not an alert change: report it under "System problems surfaced" |

## Step 4 — Propose a change and test it

For each alert that needs one, propose **one change and, at most, one alternative**: tune
(threshold, window, guard, recovery condition), re-route or demote, merge or inhibit, fix
(query, labels, target), or delete. Do not propose a change for an insufficient-data alert.

1. **Backtest.** Replay the proposed condition against the underlying metric over the
   lookback, at a step no coarser than the alert's evaluation interval, and count the
   episodes it would have produced. Emulate `for:` and multi-period evaluation: a condition
   must hold for the whole duration, not at a single sample. Report **episodes now →
   proposed**.
2. **Keep the true positives.** For every real incident in the lookback where the alert
   fired, confirm the proposed condition still fires, and report the **added detection
   delay** (a longer window delays every detection by roughly the increase). A change that
   drops a real detection is not a noise fix; say so and offer the alternative.
3. **Mind the resolution.** A backtest on rolled-up data understates spikes and overstates
   the benefit. State the resolution used. If the metric or resolution isn't available,
   label the proposal "untested" and give the reasoning instead of a count.
4. **Derive thresholds from observed values.** Quote the distribution you based it on (for
   example, p99 of the 5-minute series over 30 days outside incidents).

### Constraints to check before recommending

- **Alerts that are meant to fire.** A dead-man's switch or heartbeat alert (e.g. the
  Prometheus `Watchdog`) fires continuously by design. It is not Stuck; check that something
  is listening for its absence.
- **Alerts that drive automation.** CloudWatch alarms can trigger autoscaling policies,
  instance recovery, and failover; monitors can call webhooks, auto-remediation, or status
  pages; composite and SLO alerts can reference other alerts; inhibition rules depend on
  their source alerts. Tuning or deleting one of these changes system behavior, not just
  notifications. Find the dependents before recommending.
- **Rare-event alerts.** Disk full, certificate or token expiry, backup or replication
  failure, quota exhaustion. Never-fired is the expected state. Keep them; verify they can
  fire and that the lead time is long enough to act.
- **Sole detector.** If the alert was the first or only signal for any incident in the
  lookback, do not recommend deleting or demoting it without a replacement that would have
  fired at least as early.
- **Generated rules.** Alerts from a mixin, a vendored chart, or a shared module are changed
  through its overrides or values, not by editing generated output. Point to the override.
- **Required alerts.** Some alerts exist for compliance, contracts, or a security team. If
  you can't tell, list the alert under "Needs owner confirmation" rather than proposing its
  removal.
- **Where it changes.** Point to the rule file or IaC resource. If the alert is managed in
  the UI, say so; don't suggest UI edits to alerts that IaC would overwrite.

## Step 5 — Coverage gaps

A second, bounded pass: what should alert and doesn't. List gaps; don't design a monitoring
strategy.

- **Missed and late detections.** From incident history and postmortems in the lookback,
  find incidents first reported by a person or a customer, and incidents where the first
  alert fired well after impact began. Each is a gap with evidence.
- **Services with no user-facing symptom alert.** For each in-scope service, check for an
  alert on what users experience (errors, latency, availability, or an SLO burn rate). Having
  only cause alerts (CPU, memory, restarts) is a gap.
- **Unwatched limits.** Stateful and finite resources in scope with no alert: disk, database
  connections, queue depth or age, certificate and credential expiry, backup success.
- **Hygiene.** Paging alerts with no runbook link, no owner label, or an escalation policy
  with nobody on it.

For a proposed new alert, name the signal and confirm it exists by querying it. Derive a
starting threshold from the observed baseline, backtest it like any other proposal, and
label it a starting point.

## Parallelize

If delegation is supported, collect independent sources in parallel (definitions and routes,
firing history, paging and response history, incident history) using the host's configured
model or a suitable available model. Otherwise collect them sequentially. Keep sources
inaccessible to workers in the main session and do not pass credentials to workers. Give each
worker the exact scope, lookback as UTC timestamps, the tool or wrapper to use, and Rules 1,
2, 5, and 9 verbatim. Have them return **raw numbers** (episodes, durations, response
counts, history coverage in days) with no verdicts. The main agent classifies, proposes, and
backtests, so thresholds and judgment stay consistent.

## Step 6 — Report

```
## Scope
<teams/services, environment, alert sources, lookback (dates, UTC), conditions used,
 tools used, whether response history was available, assumptions>

## Summary
<N alerts analyzed: counts per verdict. Pages in the lookback: total, outside working hours
 (or "not computed"), share from the top 5 alerts. Projected pages if all tested changes
 were applied (or "not backtested")>

## Broken, silenced, or stuck (act first)
<alerts that cannot fire, notify no one, or are muted or ignored, with the evidence.
 These are detection gaps, not noise>

## Recommendations
| Alert | Notifies | Episodes / firing time | Median duration | Self-resolved | Response | Verdict | Pattern | Proposed change (alt) | Backtest: episodes now → proposed | Incidents still caught / added delay | Risks / notes | Where to change |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| <id and name> | <resolved route> | <retrieved values> | <retrieved value> | <share> | <strongest evidence or unknown> | <verdict> | <pattern> | <one change, one alternative> | <counts or untested> | <n of m, delay> | <dependents, unknowns> | <rule file path or UI-managed> |

## Coverage gaps
<missed or late detections with the incident, services without a symptom alert,
 unwatched limits, hygiene>

## System problems surfaced
<alerts that are noisy because something keeps breaking: the fault, how often, and the
 evidence. These are reliability work, not alert changes>

## Needs owner confirmation
<alerts that may be required, feed automation, or whose purpose you couldn't establish>

## Not checked / gaps
<alert sources not read, history shorter than the lookback, response history missing,
 proposals not backtested, alerts skipped and how they were ranked>
```

Rank recommendations by **risk first, then load removed**: detection gaps, then the changes
that remove the most pages (off-hours pages first), then marginal ones. Quote the numbers you
retrieved; don't turn them into adjectives. Recommend deletion only with the evidence that
the alert is unneeded, never because it is merely quiet.

## Install

From the root of this repository:

For Claude Code on a POSIX-like shell:

```bash
mkdir -p "$HOME/.claude/skills"
dest="$HOME/.claude/skills/alert-audit"
if [ -e "$dest" ] || [ -L "$dest" ]; then
  printf 'Already exists: %s\n' "$dest"
else
  ln -s "$PWD/alert-audit" "$dest"
fi
```

Alternatively, copy this directory into a project's `.claude/skills/` directory.
Then invoke with `/alert-audit <team | service | monitor tag | alert name | "what's paging us most">`.
