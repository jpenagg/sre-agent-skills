---
name: harden
description: >
  Defensive security review of infrastructure you own: finds misconfigurations and
  vulnerabilities across network exposure, identity/IAM, cloud resource config, Kubernetes,
  hosts, container images and dependencies, secrets, IaC, and CI/CD supply chain, then
  verifies, de-duplicates, and ranks them with evidence and remediation guidance. Tool- and
  cloud-agnostic: uses whatever is available (aws/gcloud/az, kubectl, native security
  services like Security Hub/GuardDuty/SCC/Defender, and scanners like trivy, prowler,
  checkov, kubescape, gitleaks). Use for "harden our prod account", "what's exposed to the
  internet", "audit this cluster's security", "find vulns in our infra/images". Passive and
  read-only by default; active scanning only with explicit approval.
compatibility: Requires an agent with read access to the in-scope configuration or findings through files, APIs, CLIs, or connectors. Local report writing is optional. No specific cloud, shell, scanner, or subagent support is required.
---

# Harden

You find security weaknesses in infrastructure the user owns and report them, ranked, with
evidence and fixes. You do not fix anything and you do not exploit anything. The output is a
verified findings list plus what you couldn't check.

This skill assumes **nothing** about the user's stack. Step 0 discovers which tools exist;
later steps are written as *checks* you map onto whatever tool is available. AWS and
Kubernetes examples are given because they're common. Use the equivalent for other platforms.

## Runtime adaptation

- Use the host agent's file, API, connector-discovery, and question tools. No particular
  tool name, MCP server, shell, model vendor, or delegation feature is required.
- Shell snippets assume a POSIX-like shell and illustrate provider-specific checks. Use
  equivalent APIs or native commands elsewhere. Resolve placeholders from discovery and
  verify installed versions, flags, and scanner side effects before running an example.
- Match checks to the actual architecture: cloud, on-premises, managed platform, or an
  offline repository review. Mark irrelevant layers as not applicable, with a reason.
  Missing access to an applicable layer is a gap, not a clean result.
- Use exported configuration and existing scanner findings when live access is unavailable.
  Record their timestamps and scope, and distinguish configuration risk from verified
  live exposure.

## Rules

1. **Authorized scope only.** Review only accounts, projects, clusters, repos, and IPs the
   user owns or is authorized to assess. If a target's ownership is unclear (a third-party
   SaaS, a shared/vendor IP, a domain you only *call*), leave it out and say so.
2. **Passive and read-only by default.** Use describe/list/get APIs, config files, and offline
   scanners. Never modify resources, rotate credentials, change policies, or open/close ports.
   Recommend fixes; don't apply them.
3. **Active probing needs explicit approval.** Port scans, TLS probes, and anything that sends
   traffic to a target, or runs a Job/DaemonSet in a cluster (e.g. `kube-bench`, `trivy k8s`'s
   node collector), require explicit confirmation through the host's question mechanism,
   naming the exact targets and tool. Even
   then: no exploitation, no brute force, no credential testing, no fuzzing, no load. Warn that
   probes may trigger IDS/GuardDuty alerts and suggest telling the security team first.
4. **Never invent vulnerabilities.** CVE IDs, CVSS scores, affected versions, and "known
   exploited" status come from scanner output, the provider's security service, or an
   advisory you retrieved. If you're not sure a version is affected, say "unverified".
5. **Evidence for every finding.** Each finding cites the resource ID, the exact setting or
   output that shows the problem, and how you got it. No evidence → it's a "to check", not
   a finding.
6. **Findings are confidential.** Security findings and incident details are sensitive.
   Return findings in the current user session and, when available, a local file (Step 4).
   Never post them to external chat channels,
   tickets, PRs, gists, artifacts, or web search, and don't paste resource identifiers into
   external tools. When you look up a CVE online, search the CVE ID only, never your hostnames,
   IPs, or account IDs.
7. **Redact secrets and PII.** If you find a secret, report its type, location (file:line or
   resource), and age. Never print the value (use scanners' redact flags). Credential reports
   and IAM listings contain employee names: summarize by count and role, and name a user only
   when the fix requires it.
8. **Don't install tools silently.** If a useful scanner is missing, ask before installing
   it (e.g. `brew install trivy`), and continue with what's available if the answer is no.
9. **Ask only when blocked.** Use sensible defaults (below), state them, and proceed.

## Step 0 — Discover the toolbox

```bash
# Optional discovery for a compatible shell; API connectors and offline files also work.
for c in aws gcloud az kubectl oc tsh helm terraform gh glab jq \
         trivy grype syft osv-scanner checkov tfsec kics kubescape kube-bench polaris \
         prowler scoutsuite steampipe gitleaks trufflehog semgrep zizmor actionlint \
         nmap testssl.sh sslscan; do
  command -v "$c" >/dev/null && echo "HAVE $c"
done
```

- **Connectors / MCP servers:** inspect the host's tool catalog or discovery interface for
  relevant providers and scanners. Confirm access through a read-only status call or query.
  Report authentication failures and use the host's documented login flow when needed.
- **Auth and identity:** `aws sts get-caller-identity`, `gcloud config list`, `az account show`,
  `kubectl config current-context` (or the access wrapper the environment needs, e.g.
  `tsh kubectl`, `tsh aws --app <app>`, `aws-vault exec`). Note what your identity *can't* read;
  that's a gap, not a clean result.
- **Native security services** often already have findings. Check whether they're enabled:
  AWS Security Hub, GuardDuty, Inspector, IAM Access Analyzer, Config; GCP Security Command
  Center; Azure Defender for Cloud. If they are, start from their active findings, then fill
  the gaps they don't cover.
- **Code:** locate IaC (Terraform/Helm/Kustomize/CDK/Pulumi), Dockerfiles, and CI workflows
  in the current repo and the user's code directory.

Show the user a one-line summary of available vs missing capability, by layer (Step 2).

## Step 1 — Scope

From the prompt and verified context, resolve the layers, accounts/projects/subscriptions,
regions, clusters, repositories, or on-premises targets and the environment. Default to
**passive checks within the requested scope**. Do not default to production, every account,
every region, or the current CLI context. If the target is ambiguous, ask before querying.
State the scope in one short block, including the identity and any coverage limits.

If the user named a specific target ("is the payments cluster locked down"), scope to it and
the layers that apply.

## Step 2 — Checks by layer

Treat these as a starting checklist, not a limit. Select applicable layers from the discovered
architecture and use provider equivalents; an AWS-specific feature's absence on another
platform is not a finding. Confirm each command's output shape before filtering it. Run
layers in parallel only if supported (see Parallelize).

### 2a. Network exposure (highest priority: internet-reachable is what gets attacked)

- Security groups / firewall rules allowing `0.0.0.0/0` or `::/0`, especially on admin and data
  ports (22, 3389, 3306, 5432, 6379, 9200, 27017, 2379, 10250, 5601, 8080, etc.):
  ```bash
  aws ec2 describe-security-groups --query "SecurityGroups[?IpPermissions[?IpRanges[?CidrIp=='0.0.0.0/0'] || Ipv6Ranges[?CidrIpv6=='::/0']]].{id:GroupId,name:GroupName,vpc:VpcId,perms:IpPermissions}"
  aws ec2 describe-network-interfaces --filters Name=group-id,Values=<sg> --query 'NetworkInterfaces[].{eni:NetworkInterfaceId,desc:Description,pub:Association.PublicIp}'   # is the SG actually attached?
  ```
- Public IPs and internet-facing load balancers; what's behind each listener; HTTP without
  redirect to HTTPS; TLS policy allowing TLS < 1.2; certificates near expiry.
- Data stores reachable from the internet: `PubliclyAccessible` DBs, public OpenSearch/Elastic
  domains, caches in public subnets.
- Storage exposure: account-level and bucket-level public access blocks
  (`aws s3control get-public-access-block --account-id <id>`), bucket policies with
  `"Principal": "*"`, public snapshots/AMIs
  (`aws ec2 describe-snapshots --owner-ids self --restorable-by-user-ids all`,
  `aws ec2 describe-images --owners self --filters Name=is-public,Values=true`, and for each
  manual RDS snapshot `aws rds describe-db-snapshot-attributes --db-snapshot-identifier <id>`,
  where `restore` containing `all` means public and other account IDs mean shared).
- Kubernetes API endpoint: public access and its allowed CIDRs
  (`aws eks describe-cluster --name <c> --query 'cluster.resourcesVpcConfig'`).
- Kubernetes: `LoadBalancer`/`NodePort` Services and Ingresses/Gateways exposing internal
  apps; namespaces with no NetworkPolicy; service mesh mTLS in `PERMISSIVE` mode
  (`kubectl get peerauthentication -A`).
- Flow logs disabled on VPCs that hold production workloads.

### 2b. Identity and access

- Root/owner account: MFA on, no access keys. IAM users with console access but no MFA,
  access keys older than 90 days or unused for 90+ days, using the organization's policy
  where available instead of treating those example ages as universal requirements.
  On AWS, prefer an existing credential report (`aws iam get-credential-report`). Decode
  and summarize locally without printing the raw report. Generating a fresh report is a
  separate operation; if required, explain it and obtain approval or record the gap.
  Summarize by count; see Rule 7.
- Over-broad policies: `Action: "*"` with `Resource: "*"`, `iam:PassRole` on `*`, wildcard
  `s3:*`/`kms:*` on production data. Prefer IAM Access Analyzer findings when available
  (`aws accessanalyzer list-analyzers`, `list-findings`).
- Trust policies: `Principal: "*"` or `AWS: "*"` without conditions; cross-account trust to
  accounts not in the user's org (list them; the user decides which are expected); CI OIDC
  roles whose `sub` condition is a wildcard (e.g. `repo:<org>/*` lets any repo in the org
  assume a prod role) or missing entirely.
- Kubernetes RBAC: bindings to `cluster-admin`, wildcard verbs/resources, `system:anonymous` /
  `system:unauthenticated` bindings, service accounts with `secrets` `get/list` cluster-wide,
  default service account tokens automounted in workloads that don't need them.
- Workload identities (IRSA / Workload Identity / managed identity) scoped wider than the
  workload needs.

### 2c. Cloud resource configuration

- Encryption at rest: verify effective encryption and key access using the provider's
  current behavior. Examples include EBS default encryption (`aws ec2 get-ebs-encryption-by-default`),
  RDS `StorageEncrypted`, and storage or snapshot encryption. An absent explicit setting
  does not establish lack of encryption when the platform encrypts by default. Evaluate
  wildcard key policies with their conditions and provider-specific authorization semantics.
- Logging and detection: CloudTrail multi-region and actually logging
  (`aws cloudtrail describe-trails`, `get-trail-status`), GuardDuty enabled in every used region,
  Config recording, or equivalent centralized services. Confirmed absence of needed
  detection is a finding; denied access, delegated administration, or an unknown setting
  is a gap until verified.
- Backups: production databases without backups, or with backups shared/public.
- Instance metadata: IMDSv1 still allowed
  (`aws ec2 describe-instances --query 'Reservations[].Instances[].{id:InstanceId,tokens:MetadataOptions.HttpTokens,hop:MetadataOptions.HttpPutResponseHopLimit}'`;
  `optional` = IMDSv1 on).
- Run a CSPM scanner if available: `prowler aws --severity critical high` (read-only), or
  Steampipe/ScoutSuite. Treat their output as candidates and verify the important ones.

### 2d. Kubernetes workloads and cluster

- Pod security: `privileged`, `hostNetwork`/`hostPID`/`hostIPC`, `hostPath` mounts (especially
  `/`, `/var/run/docker.sock`, `/var/run/containerd`), `allowPrivilegeEscalation`, running as
  root, added capabilities (`SYS_ADMIN`, `NET_ADMIN`), no `readOnlyRootFilesystem`:
  ```bash
  kubectl get pods -A -o json | jq -r '.items[] | . as $p | .spec.containers[] |
    select(.securityContext.privileged==true or .securityContext.allowPrivilegeEscalation==true
           or ($p.spec.hostNetwork==true) or ($p.spec.hostPID==true)) |
    [$p.metadata.namespace, $p.metadata.name, .name] | @tsv'
  ```
  Expect some legitimate system DaemonSets (CNI, CSI, log/metrics agents). List them separately
  as "expected privileged" instead of mixing them with app findings.
- Pod Security admission labels on namespaces (`kubectl get ns --show-labels`), or the policy
  engine in use (Kyverno/Gatekeeper) and whether it enforces or only audits.
- Secrets: credentials stored directly in ConfigMaps or workload configuration. Inspect
  references and storage settings without retrieving secret values. For encryption at rest,
  check effective platform defaults and cluster version; absence of an optional
  customer-managed key setting does not by itself mean secrets are unencrypted.
- Images: `:latest` tags, registries outside the org's allowed list, images not pinned by
  digest for critical workloads.
- Cluster version and add-ons at or past end of support.
- Scanners if available: Kubescape, Polaris, kube-bench, or Trivy. Check the installed
  version's behavior and flags first: some modes create workloads, collect host data, send
  probes, or upload findings. Use local-output, passive modes by default; node collectors
  and other active modes require approval under Rule 3.

### 2e. Hosts and OS

- End-of-life OS/images and image age; patch compliance (e.g. `aws ssm list-compliance-summaries`
  or Inspector findings); SSH/RDP reachable from outside. A missing management agent is a
  coverage question, not proof that a host is unpatched or unmonitored: check for alternate
  management, immutable-image replacement, or provider-managed hosts.

### 2f. Container images and dependencies

- Get the images **actually running** (from the cluster or task definitions), dedupe, and scan:
  ```bash
  kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u
  trivy image --severity HIGH,CRITICAL --ignore-unfixed --format json <image>   # or grype <image>
  ```
  Registry auth may be needed. If pulling isn't possible, use the registry's own scan results
  (e.g. ECR `describe-image-scan-findings`) or Inspector.
- Repos: `osv-scanner` / `trivy fs` on lockfiles; open Dependabot/code-scanning alerts via
  `gh api repos/<org>/<repo>/dependabot/alerts` if accessible.
- **Prioritize by exploitability**, not just CVSS. A fix exists, the package is actually loaded,
  it's internet-facing, and it's in CISA's Known Exploited Vulnerabilities catalog: download the
  catalog (`curl -s https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`)
  and match CVE IDs locally. Don't send your CVE list anywhere.

### 2g. Secrets in code and config

- `gitleaks detect --source <repo> --redact` (or `trufflehog git file://<repo> --only-verified`
  only with approval: verification contacts the provider). Check IaC variables, Helm values,
  CI workflow env, Dockerfiles, and `.env` files committed to git.
- Don't read Terraform state or secret stores to "check" secrets. State contains secrets in
  plaintext. Note where state lives and whether its bucket/backend is locked down instead.

### 2h. IaC and CI/CD supply chain

- IaC misconfig scan: `trivy config <dir>` or `checkov -d <dir>`. Map findings to the live
  resources in 2a–2d. A finding in IaC that's also live ranks higher than an IaC-only one.
- GitHub Actions (or equivalent): `pull_request_target` or `workflow_run` workflows that check
  out PR code, missing or broad `permissions:` (default write-all), third-party actions not
  pinned to a full commit SHA, script injection from `${{ github.event.* }}` into `run:`,
  self-hosted runners on public repos, long-lived cloud keys in secrets where OIDC would do.
  `zizmor` / `actionlint` if available.
- Branch protection on default branches of repos that deploy to production
  (`gh api repos/<org>/<repo>/branches/<b>/protection`).

## Step 3 — Verify, de-duplicate, rank

1. **Verify.** For each high/critical candidate, confirm it's live and reachable as claimed. Is
   the SG attached to anything? Is the resource running? Is the "public" bucket actually
   blocked at the account level? Is there a compensating control (WAF, private subnet with no
   route, IP allowlist upstream)? Downgrade or drop, and say why.
2. **De-duplicate** the same issue reported by multiple tools or on multiple members of one
   group. Report it once, with a count.
3. **Rank** by exposure × impact × exploitability:

| Severity | Typical examples |
|---|---|
| **Critical** | Internet-reachable admin/data port; publicly readable production data/snapshots; known-exploited CVE in an internet-facing service; valid credential committed to a repo; any-principal role trust |
| **High** | Admin-equivalent IAM without MFA; IMDSv1 on internet-facing hosts; privileged app pods; fixable critical CVEs in running images; CI OIDC wildcard to prod; CloudTrail/GuardDuty off |
| **Medium** | Unencrypted at rest; no NetworkPolicy in sensitive namespaces; permissive mTLS; stale access keys; EOL OS internal-only |
| **Low / Info** | Hardening best practices with little direct exposure; unpinned actions in low-privilege workflows |

Adjust for context and say why (e.g. "public bucket, but it's the static site's asset bucket:
Info"). When you can't tell whether something is intended, mark it **"Needs owner
confirmation"** rather than guessing.

## Parallelize

If the host supports delegation, run independent layer groups (network+cloud config,
identity, Kubernetes, images+deps, code+CI) in parallel using the configured model or an
appropriate available model. Otherwise run them sequentially. Do not require a specific
model or pass credentials to workers; keep sources unavailable to workers in the main
session. Give each worker the exact scope, identity/wrapper, its checklist, and Rules 2–7
verbatim. Have them return **raw
candidates with evidence** (resource ID, setting, command used), not severities. The main agent
verifies, de-duplicates, and ranks so the bar stays consistent.

## Step 4 — Report

Return the report in the current user session. If local file writing is available, use the
user's requested path or the host's documented private scratch directory and report the
path. If neither exists, ask for a path or keep the report in-session and note that no file
was saved. Do not assume a scratch directory exists, silently choose a synced/public
location, commit the report, or publish it externally (Rule 6).

```
## Scope
<accounts/projects, regions, clusters, repos, identity used, passive/active, tools used, date (UTC)>

## Summary
<counts by severity; top 3 things to fix first, one line each>

## Findings
| # | Severity | Layer | Resource | Issue | Evidence | Fix | Where to change |
|---|---|---|---|---|---|---|---|
| 1 | Critical | Network | sg-0abc (attached to 3 ENIs) | 5432 open to 0.0.0.0/0 | describe-security-groups rule; DB in public subnet | Restrict to app SG; set PubliclyAccessible=false | terraform/rds/sg.tf:20 |

## Needs owner confirmation
<items that may be intentional: public buckets, cross-account trusts, exposed services>

## Expected / accepted
<privileged system DaemonSets, intentionally public endpoints, with the reason>

## Not checked / gaps
<layers or accounts you couldn't read, missing scanners, active checks not approved, images you couldn't pull>
```

Fixes are recommendations. Point to the IaC that owns each resource so the change goes
through review. Don't suggest console edits that would drift from IaC. Remind the user to
verify findings and remediation priorities with their security team before acting, and to
handle the report as confidential.

## Install

From the root of this repository:

For Claude Code on a POSIX-like shell:

```bash
mkdir -p "$HOME/.claude/skills"
dest="$HOME/.claude/skills/harden"
if [ -e "$dest" ] || [ -L "$dest" ]; then
  printf 'Already exists: %s\n' "$dest"
else
  ln -s "$PWD/harden" "$dest"
fi
```

Alternatively, copy this directory into a project's `.claude/skills/` directory.
Then invoke with `/harden <account | cluster | repo | "what's exposed to the internet">`.
