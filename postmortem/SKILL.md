---
name: postmortem
description: >
  Drafts a blameless incident postmortem from the evidence that already exists: an
  investigation report, the incident channel or ticket, paging history, alerts, deploy
  history, and metrics. Tool- and platform-agnostic: uses whatever is available (any CLI or
  MCP server: Slack/Teams, PagerDuty/Opsgenie/incident.io, Jira/Linear/GitHub Issues,
  Datadog, Prometheus/Grafana, CloudWatch, Argo CD/Flux/Helm, git). Rebuilds the timeline
  from data, checks the team's narrative against it, measures impact and time to detect and
  mitigate, separates trigger from contributing factors, and proposes specific action items
  tied to evidence. Use for "write the postmortem for yesterday's checkout outage", "draft an
  incident review from this channel", "turn this investigation into a postmortem", "RCA doc
  for INC-123". Read-only; the draft stays local for human review.
compatibility: Requires an agent with read access to the incident record through files, APIs, CLIs, or connectors. Local draft writing is optional. No specific incident tool, chat platform, shell, MCP server, or subagent support is required.
---

# Postmortem

You turn an incident's evidence into a blameless written review. You do not fix things, file
tickets, or publish anything. The output is a draft postmortem in which every statement is
either backed by data you retrieved or labeled as a report from a person, plus the questions
only the people involved can answer.

A postmortem is not a second investigation. If an investigation report exists, start from it.
If the cause was never established, say so in the draft instead of writing a confident story;
recommend an investigation first (the `investigate` skill in this repository, if installed).

This skill assumes **nothing** about the user's stack. Step 0 discovers which tools exist;
later steps are written as *operations* ("read the paging history", "re-run the alert query")
that you map onto whatever tool is available.

## Runtime adaptation

- Use the host agent's available tools for reading files, querying APIs, searching connectors,
  and asking questions. Tool names, discovery mechanisms, permissions, and login flows vary.
  MCP servers, shell access, persistent memory, and subagents are optional.
- Shell snippets are examples for POSIX-like shells, not commands to run verbatim. On other
  systems use equivalent tools or APIs. Resolve placeholders from retrieved evidence; check
  installed versions and supported flags before using provider-specific examples.
- Work from supplied exports (a chat export, a pasted investigation report, dashboard
  screenshots, a ticket dump) when live access is unavailable. Label their capture times and
  coverage. A statement you could not check against data stays "Reported".
- Postmortems are usually written days later. Expect rolled-up metrics, expired events, and
  trimmed deploy history; record the resolution and retention you actually had.

## Rules

1. **Read-only, and draft-only.** Never post to chat, create or edit tickets, update the
   incident tool, edit a wiki or status page, or change any system. Never commit the draft
   or open a pull request unless the user asks. Recommend action items; don't file them.
2. **Blameless.** Describe what the system allowed and what people could see at the time.
   Refer to people by role ("on-call", "a responder", "the change author"), never by name.
   "Human error" is a starting point, not a cause: ask what made the action possible, easy,
   or look correct. No "should have", "failed to", "careless", or "obvious".
3. **Claims are not evidence.** The channel's conclusion, the ticket's root-cause field, and
   AI-generated summaries are reports to check against data. Label every timeline row and
   causal statement **Verified** (data you retrieved), **Reported** (someone said it,
   unchecked), or **Inferred** (your reasoning from verified facts).
4. **Never invent numbers or names.** Impact figures, customer counts, durations, SLO
   targets, service names, and ticket IDs come from a source you retrieved. If a figure is
   unavailable, write "unknown" and what would provide it. Never estimate revenue.
5. **Empty is not "nothing happened."** Empty results may mean expired auth, the wrong
   context/account/region, wrong labels, retention limits, or a truncated result. Check those
   before writing that something did not occur.
6. **Don't overstate the cause.** State how well the cause is established (Step 3). A draft
   that says "probable" where the evidence is probable is more useful than a tidy story.
7. **Protect personal and sensitive data.** Summarize log patterns and counts; don't paste
   raw lines with emails, names, IDs, or tokens. Don't name customers unless the user asks.
   If the incident is a security incident (suspected breach, leaked credential, data
   exposure), stop and ask how it should be handled before drafting; those reviews usually
   have restricted distribution.
8. **UTC everywhere** in the output, with the user's timezone in parentheses for key moments
   when known. Do not assume the agent's machine timezone is the user's timezone.
9. **Ask only when blocked.** If you can't tell which incident is meant, or it is still
   ongoing, ask. Otherwise state your assumptions and proceed.

## Step 0 — Discover the toolbox and the inputs

Find out what you can read **before** planning. Inspect available connectors, files, and
CLIs. Run the following discovery only if a compatible shell is available.

```bash
# CLIs present? (only report what exists; don't install anything without asking)
for c in gh glab jq git pd kubectl oc tsh helm argocd flux aws gcloud az pup logcli \
         promtool sentry-cli; do command -v "$c" >/dev/null && echo "have: $c"; done
```

- **Connectors / MCP servers:** use the host's tool catalog or search mechanism, if available,
  to discover each category below. Confirm access with a read-only status call or query. An
  authentication error means that source is unavailable until the user completes the host's
  login flow; do not assume a particular tool name or slash command exists.
- **Auth checks:** run a cheap identity/status call for each tool you plan to use. If an
  access wrapper is required (Teleport, SSO-backed profiles, a bastion), use the documented
  one or ask; do not source arbitrary shell startup files.

| Category | Used for | Examples (use what's present) |
|---|---|---|
| Investigation report | Evidence already collected, hypotheses, gaps | Earlier output in this session, a file, a linked doc |
| Human context | Decisions, actions, theories, who knew what when | Slack / Teams incident channel, call notes, ticket comments |
| Incident tool / paging | Declared, acknowledged, escalated, resolved times; severity | PagerDuty, Opsgenie, incident.io, FireHydrant, Alertmanager |
| Metrics / logs / traces | Impact start and end, magnitude, checking claims | Datadog, Prometheus/Grafana, New Relic, CloudWatch, Loki, Sentry |
| Deploys / config | What changed, what the mitigation actually was | Argo CD, Flux, Helm history, CI/CD runs, deploy-repo git log, flag tool |
| Past postmortems | Template, recurrence, earlier action items | `docs/postmortems/`, wiki, incident tool history |
| Issue tracker | Existing tickets for proposed actions | Jira, Linear, GitHub / GitLab issues |

**Missing categories:** say which are missing and what the draft loses without them (no
metrics access means impact and recovery are Reported, not Verified). Offer the options
(connect, log in, supply an export, or skip), continue with what you have, and carry the gap
into the draft. Never fabricate values for a missing source.

## Step 1 — Establish the incident record

Resolve exactly one incident and state it in one short block.

| Field | Take from | If missing |
|---|---|---|
| Incident | ID, channel, ticket, or investigation report named in the request | Ask; do not pick "the most recent one" |
| Status | Incident tool state plus the symptom's current value | If impact is ongoing, stop: offer a status summary instead and say a postmortem is premature |
| Severity | As declared by the team | Write "not declared"; do not assign one |
| Affected services and environment | The alert's scope labels, the incident record, verified contexts | Ask if ambiguous; do not default to production |
| Window | Earliest symptom in data − 30m to recovery + 30m | Derive from the alert query (Step 2), not from the declared times |
| Audience and template | The user's request; an existing template or recent postmortem in the repo or wiki | Use the default template in Step 6 and say so |

**From an investigation report:** rows that cite a query or command and its result are
evidence; reuse them and keep their sources. Its hypotheses stay hypotheses until Step 3.
Carry its "Not checked / gaps" forward rather than dropping them.

**From an incident channel or ticket** (read-only tools only):
1. Find the channel by name, including archived ones. Read it **oldest-first**; the opening
   messages and pins carry the summary, severity, services, and links.
2. Follow forwarded messages, threads, and cross-channel links one level deep. The key
   analysis is often in a thread or a separate channel.
3. Harvest: alert and dashboard links, status milestones, **actions taken** (rollback,
   restart, scale-up, failover, flag flip, merged PR) with who-by-role and when, **decisions
   and what they were based on**, theories proposed and dropped, and things people said
   they could not find or access.

**Team template:** if the organization has a postmortem template or a directory of past
postmortems, conform to its headings and add any sections from Step 6 it lacks at the end
rather than discarding content.

## Step 2 — Build the timeline from two sources

Collect the **system record** (what the data shows) and the **human record** (what people
saw, said, and did) separately, then merge them. Disagreements between the two are findings.

### Key times

Each milestone has a preferred source. Record the time, the source, and its status.

| Milestone | Preferred source | Trap |
|---|---|---|
| Triggering change or event | Deploy/sync event, flag audit log, cloud event, traffic or dependency metric | Merge time is not deploy time; pair commits with rollout events |
| Impact start | The alert's own query (or the user-facing SLI) crossing out of baseline | Not the alert's trigger time: evaluation windows and `for:` delays lag the real start |
| Detected | First alert trigger, or first human report if it came earlier | A customer report or a colleague's message before the first page is a detection gap; note which came first |
| Acknowledged | Paging tool log | Auto-acknowledge and escalation to a second responder look alike; read the log entries |
| Declared | Incident tool or channel creation | Channel creation can precede or follow the declaration |
| Mitigating action | Deploy tool, cloud audit log, orchestration events | The chat message saying "rolled back" is when it was typed, not when it took effect |
| Impact end | The same query returning to baseline and staying there | Not when the incident was marked resolved |
| Resolved (declared) | Incident tool | Often minutes to hours after impact ended |

From these compute: **time to detect** (impact start → detected), **time to acknowledge**,
**time to mitigate** (detected → impact end), and **total impact duration**. If a milestone
is only Reported, the intervals that depend on it are approximate; say so.

### Timeline traps

- **Message time is not event time.** Verify actions against the system that performed them.
- **Timezones in chat and exports.** Exports may be in the exporter's local time or in epoch
  seconds; screenshots carry the viewer's timezone. Confirm before converting to UTC.
- **Flapping and reopened incidents.** If the symptom recovered and returned, record each
  episode's start and end rather than one span, and explain the quiet stretches.
- **Rolled-up metrics.** At 5-minute or hourly resolution a start time is a range, not a
  minute. Write the range.
- **Short-retention sources.** Kubernetes events, previous-container logs, and capped deploy
  history may already be gone. Current state cannot establish what happened then; list it as
  a gap.
- **Decisions belong in the timeline with their inputs.** "Chose restart over rollback;
  dashboard at the time showed memory climbing" is more useful than "restarted".

## Step 3 — Check the narrative

Before writing, test the story the team settled on against the data.

1. **Did the mitigation end the impact?** Compare the mitigating action's effective time with
   impact end. Recovery that began *before* the action, or long after it, means something
   else ended the impact. A rollback also restarts processes, resets connections, and flushes
   caches, so recovery after a rollback does not by itself show the reverted change was
   the cause.
2. **Does the trigger precede the impact, every time?** For multi-episode incidents, check
   each episode. If the symptom also appears in a clean baseline window without the trigger,
   the trigger is not sufficient on its own.
3. **Does the mechanism have evidence, or only timing?** Look for the signal the proposed
   mechanism predicts (pool saturation, OOM kills, a specific error class, a dependency's
   latency). Timing alone is correlation.
4. **What does not fit?** Unexplained observations go in the draft, not in the bin.
5. **Check each harvested claim** you rely on. Unsupported ones stay Reported, or are listed
   as checked and not supported.

Then state the cause at one of three levels, in these words:

| Level | Meaning |
|---|---|
| **Established** | The mechanism was demonstrated: reproduced, or the data shows each link from trigger to symptom, and removing the trigger removed the symptom |
| **Probable** | Timing and mechanism fit, nothing contradicts it, but a link is undemonstrated. Name the missing link |
| **Not established** | Competing explanations remain, or the evidence is gone. List the candidates and what would distinguish them |

Keep verification proportionate: re-run the alert query, confirm the key times, and test the
mitigation. If the cause needs real investigation, stop expanding, mark it "Not established",
and recommend an investigation as the first action item.

## Step 4 — Measure impact

Report what users experienced, with the basis for each figure.

| Measure | How | Trap |
|---|---|---|
| Duration | Impact start → impact end, per episode | Declared times overstate or understate it |
| Magnitude | Failed or degraded requests ÷ total, from the user-facing SLI | A partial outage is not "down"; give the percentage and the peak |
| Who was affected | Breakdown by endpoint, region, tenant tier, client type | Distinct affected users needs a per-user field; if absent, write "unknown" |
| Hidden impact | Client retries, queued work, delayed jobs, dropped events, stale data | Successful retries hide user-visible failures; backlogs extend impact past recovery |
| Data integrity | Lost, duplicated, or corrupted writes | "No data loss" needs a check that was run, not an absence of complaints |
| Error budget | Budget consumed against a **defined** SLO | If no SLO exists, say so; do not invent a target |
| Support and external signal | Ticket counts, status page updates | Use counts from the support tool; don't infer from chat |

Mark each figure **measured**, **estimated** (state the method), or **unknown**.

## Step 5 — Analyze

Sort the verified facts into five parts. Each part cites timeline rows.

| Part | Question |
|---|---|
| Trigger | What change or event started this, and why at this moment? |
| Contributing factors | What conditions let the trigger become user impact? Which defense was absent, bypassed, or ineffective: tests, review, canary, limits, timeouts, isolation, capacity headroom? |
| Detection | What told us, how long after impact start, and was it the signal we would want? |
| Response | What helped or slowed diagnosis and mitigation: access, runbooks, dashboards, ownership, escalation, the mitigation's own duration? |
| Recovery | What ended the impact, and what work remained afterwards (backlog, cleanup, data repair)? |

- **Several factors, not one root cause.** If the team's template has a single "root cause"
  field, fill it with the trigger and the main condition that made it harmful, and keep the
  full list under contributing factors.
- **Ask "what made this possible" until the answer is a property of the system.** Stop when
  the next answer would be speculation; write that as an open question.
- **Record what went well and where luck helped.** A fast rollback, a responder who happened
  to be online, an incident at low traffic. Luck is a contributing factor that pointed the
  other way.
- **Don't guess intent.** Why someone chose an action goes under open questions unless they
  said why at the time.

Blameless phrasing describes the system the person was working in:

| Instead of | Write |
|---|---|
| "The engineer forgot to run the migration" | "The deploy proceeded with a pending migration; nothing in the pipeline checks for one" |
| "On-call was slow to respond" | "The page was acknowledged 14m after it fired, at 03:10 local for the responder, after one escalation" |
| "This should have been caught in review" | "The review showed the template change; the rendered config, where the behavior change was visible, was not part of the diff" |

### Prior incidents

Search past postmortems and incident history for the same service, alert, or factor. For each
match report whether its action items were completed (ticket state, or whether the control
now exists). An open action item from an earlier postmortem that would have applied here is
a contributing factor; say so plainly. If you could not search, list it as a gap.

## Step 6 — Action items and draft

### Action items

Every action item addresses a numbered contributing factor or a detection/response gap.

- **Specific and checkable.** State what will exist when it is done.
- **Typed:** *Prevent* (stops the trigger or the factor), *Detect* (shortens time to detect),
  *Mitigate* (limits blast radius or shortens recovery), *Process* (ownership, runbook,
  escalation).
- **Prefer the class over the instance.** An item that only prevents this exact trigger is
  weaker than one that removes the condition; if you propose the narrow one, say so.
- **Quantify only from data.** "Would have fired at 14:02, 18m before the first page" is
  allowed when you ran the proposed alert expression over the incident window and the
  baseline. Otherwise don't claim a saving.
- **Suggest an owning team or role, never a person.** Write "suggested" so the review
  assigns it.
- **Check the tracker** for an existing ticket and link it instead of proposing a duplicate.
- **Rank them** and keep the list short. Name the single item that would have reduced impact
  most. Drop "be more careful", "improve monitoring", and "add documentation" unless made
  specific.

| Weak | Stronger |
|---|---|
| "Improve monitoring" | "Alert when connection-pool use on the orders database exceeds 90% for 5m; over the incident window this crosses at 14:02" |
| "Be careful with config changes" | "Route changes to the shared proxy config through the canary stage that application deploys already use" |
| "Update the runbook" | "Add the failover command and its expected duration to the runbook linked from the alert; the responder searched for it for 9m" |

### Draft

Return the draft in the current user session. If local file writing is available, write it
to the user's requested path, or to the repository's existing postmortem directory if the
user asks for that, and report the path. Do not commit it, publish it, or post it (Rule 1).

```
# Postmortem: <what happened, stated as system behavior> — DRAFT
<incident ID · severity as declared · date (UTC) · status: agent-drafted, not yet reviewed>

## Summary
<3–5 sentences: what users experienced and for how long, the trigger, what ended it,
 and how well the cause is established>

## Impact
| Measure | Value | Basis (measured / estimated / unknown) |
|---|---|---|

## Key times (UTC)
| Milestone | Time | Source | Status |
|---|---|---|---|
Time to detect · time to acknowledge · time to mitigate · total impact duration

## Timeline (UTC)
| Time | Source | Event | Status |
|------|--------|-------|--------|

## What happened
<trigger → mechanism → impact → recovery. Cause: Established / Probable / Not established,
 with the missing link or the competing candidates. What does not fit.>

## Contributing factors
1. <condition> — evidence: <timeline rows>

## Detection and response
<how it was found and by what; what helped; what slowed diagnosis or mitigation>

## What went well / where we got lucky

## Action items
| # | Action | Type | Addresses | Done when | Suggested owner | Priority | Existing ticket |
|---|---|---|---|---|---|---|---|

## Prior incidents
<matches and the state of their action items, or "none found" / "not searched">

## Open questions for review
<what only the people involved can answer: reasons for decisions, what was on screen,
 Reported statements that need confirmation>

## Not verified / gaps
<missing tools, auth, retention, rolled-up or truncated data, claims left as Reported>
```

Report rules:
- Every causal statement and contributing factor cites timeline rows. No evidence, no factor.
- Quote the numbers you retrieved; don't turn them into adjectives.
- Say plainly where the data disagrees with the team's conclusion or the declared times.
- Roles, not names. System properties, not personal failings.
- Keep "DRAFT" and the unreviewed status until a person removes them. Tell the user which
  sections most need review by the people who were there.
- **External or customer-facing summary:** write one only if asked, as a separate section.
  Include what customers experienced, when, and what is being done. Leave out internal
  hostnames, people, vendor attribution, and anything not Verified.

## Parallelize

If the host supports delegation, collect independent sources in parallel for larger
incidents: the human record (channel, ticket), paging and incident-tool history, the system
record (alert query, impact metrics, deploy history), and past postmortems with their action
items. Otherwise collect them sequentially. Use the configured model or an appropriate
available one; do not require a specific model or pass credentials to workers, and keep
sources unavailable to workers in the main session. Give each worker the incident scope, the
window as Unix seconds, and Rules 1–5 and 7 verbatim. Have them return **raw observations
with sources, exact values, and UTC timestamps**, not causes or blame. The main agent merges
the timeline, checks the narrative, and writes the analysis so the bar stays consistent.

## Install

From the root of this repository:

For Claude Code on a POSIX-like shell:

```bash
mkdir -p "$HOME/.claude/skills"
dest="$HOME/.claude/skills/postmortem"
if [ -e "$dest" ] || [ -L "$dest" ]; then
  printf 'Already exists: %s\n' "$dest"
else
  ln -s "$PWD/postmortem" "$dest"
fi
```

Alternatively, copy this directory into a project's `.claude/skills/` directory.
Then invoke with `/postmortem <incident ID | incident channel | investigation report | description>`.
