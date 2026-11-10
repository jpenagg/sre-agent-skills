# SRE Agent Skills

Reusable agent workflows for incident investigation, infrastructure security reviews, and right-sizing.

Each skill is a `SKILL.md` file with instructions for gathering evidence, checking it in context, and reporting findings. The workflows use the tools available in your environment, with AWS and Kubernetes examples throughout.

## Skills

| Skill | What it does | Output |
| --- | --- | --- |
| [investigate](investigate/SKILL.md) | Correlates alerts, metrics, logs, deploys, and incident discussions. | UTC timeline, ranked hypotheses, and evidence gaps. |
| [harden](harden/SKILL.md) | Reviews network exposure, identity, cloud configuration, Kubernetes, images, and CI/CD. | Verified, prioritized findings with remediation guidance. |
| [right-size](right-size/SKILL.md) | Compares historical utilization with provisioned capacity and candidate sizes. | Resource recommendations with verified specs, prices, and constraints. |

## Requirements

- An agent that can load `SKILL.md` instructions and read relevant evidence through files, APIs, CLIs, or connectors. Shell access is optional.
- Authenticated access to the systems you want to inspect: cloud CLIs, Kubernetes, observability tools, source repositories, or MCP servers.
- For right-sizing, at least 14 days of utilization data; the default lookback is 30 days.

You do not need every tool mentioned in the skills. Each workflow starts with tool and access discovery and reports missing capabilities. No credentials, scanners, or MCP servers are bundled.

The skill metadata uses `name`, `description`, and `compatibility`; the workflows do not require named agent tools or model vendors. MCP discovery, permissions, and optional delegation depend on the host. If subagents are unavailable, collect independent evidence sources sequentially. Supplied exports can substitute for live tools when their time range and coverage are sufficient.

The installation and slash-command examples below are for **Claude Code on a POSIX-like shell**. For another host, use its documented skill-loading mechanism or provide the relevant `SKILL.md` as instructions. Shell snippets inside the skills are provider-specific examples to adapt, not a cross-platform installer or a claim of tested support for every provider.

## Install in Claude Code

From the root of your local checkout, link the skills into your personal skills directory:

```sh
mkdir -p "$HOME/.claude/skills"
for skill in investigate harden right-size; do
  dest="$HOME/.claude/skills/$skill"
  if [ -e "$dest" ] || [ -L "$dest" ]; then
    printf 'Already exists: %s\n' "$dest"
  else
    ln -s "$PWD/$skill" "$dest"
  fi
done
```

These commands skip existing installations. Keep the checkout in place because the links point to it. Changes in the checkout are available through the same links.

For a project-specific installation, copy the desired skill directories into that project's `.claude/skills/` directory instead. Each installed directory must contain its `SKILL.md` file.

## Usage

Provide the target, environment, and time range when relevant. For example:

```text
/investigate Why did checkout latency spike in staging around 14:20 UTC today?
/harden Review the staging Kubernetes cluster's security configuration, passive checks only.
/right-size Review the orders database in staging using the last 30 days of metrics.
```

An alert URL, incident channel, resource identifier, or repository path can provide the starting scope. The agent should confirm the actual account, context, and available data before drawing conclusions.

## How they work

1. Discover tools and confirm access and scope.
2. Retrieve observations from the available sources.
3. Check candidates against context, baseline behavior, and constraints.
4. Report evidence, recommendations, and anything that could not be verified.

`investigate` and `right-size` are read-only workflows. `harden` defaults to passive review; active probes require explicit approval, and reports stay local. These are instructions for the agent, not a permissions sandbox—use the host's permissions and appropriately scoped credentials to enforce access boundaries.

The workflows recommend changes; they do not apply remediations. AWS examples and default thresholds are starting points to interpret in the context of your systems.

## Repository layout

```text
investigate/SKILL.md
harden/SKILL.md
right-size/SKILL.md
```

Each skill is self-contained. There are no required supporting scripts or build steps.

## Background

[Turning SRE Workflows into Agent Skills](https://jpena.dev/posts/sre-agent-skills/)

## License

[MIT](LICENSE)
