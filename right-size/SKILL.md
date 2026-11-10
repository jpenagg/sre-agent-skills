---
name: right-size
description: >
  Finds over- and under-provisioned infrastructure (VMs/EC2, autoscaling groups, managed
  databases like RDS/Aurora/Cloud SQL, caches like ElastiCache/Redis, search/streaming
  clusters, Kubernetes nodes and pod requests/limits) by comparing real utilization over a
  representative lookback against provisioned capacity, then recommends a better-fitting
  instance type or size with verified specs, price delta, and risks. Tool- and
  cloud-agnostic: uses whatever is available (aws/gcloud/az, CloudWatch, Datadog,
  Prometheus, kubectl, cloud-native recommenders, IaC repos). Use for "is X oversized",
  "right-size our RDS instances", "which EC2s are underused", "what instance type should Y
  be", "are these pods' requests too high". Read-only.
compatibility: Requires an agent with read access to capacity specifications and historical utilization through files, APIs, CLIs, or connectors. No specific cloud, shell, MCP server, or subagent support is required.
---

# Right-Size

You measure how much of each resource is actually used, compare that to what it's provisioned
with, and recommend a better fit. You do not resize anything. The output is a per-resource
table of verdicts and recommended types, each backed by numbers you retrieved, plus what you
couldn't check.

This skill assumes **nothing** about the user's stack. Step 0 discovers which tools exist;
later steps are written as *operations* ("get p95 CPU", "list instance specs") that you map
onto whatever tool is available. AWS examples are given because they're common. Use the
equivalent for other clouds or on-premises systems.

## Runtime adaptation

- Use the host agent's available file, API, connector-discovery, and question tools. No
  particular tool name, MCP server, model vendor, or delegation feature is required.
- Shell snippets assume a POSIX-like shell and illustrate provider-specific operations.
  Use equivalent native commands or APIs elsewhere. Resolve placeholders from discovery;
  verify command versions, supported flags, units, and response fields before use.
- For on-premises or custom capacity, use verified hardware/VM quotas and user-provided
  cost data instead of inventing an equivalent cloud SKU. If dollars cannot be priced,
  report capacity changes and mark financial savings as unverified.
- Supplied inventory and metric exports can substitute for live tools. Record their capture
  time, lookback, resolution, and gaps, and limit conclusions to that evidence.

## Rules

1. **Read-only.** Never modify, resize, stop, reboot, or scale resources, and never edit IaC or
   apply/plan-apply. Recommend changes and point to where they'd be made. If the user later
   asks for a PR, that's a separate, explicit request.
2. **Never invent specs or prices.** vCPU, memory, network bandwidth, and price for a type come
   from the provider's API (e.g. `aws ec2 describe-instance-types`, `aws pricing get-products`)
   or retrieved provider documentation or records the user provides. If you couldn't retrieve
   a price, write "price not retrieved", not an estimate from memory.
3. **Never invent names.** Resource IDs, metric names, tags, accounts, and regions come from
   discovery commands, IaC, or the user. Confirm a metric exists by querying it and getting
   data back.
4. **Peaks, not averages.** Decide on p95/p99 and max over the lookback, not the mean. A
   database that averages 10% CPU but hits 90% during a nightly batch job is not oversized.
5. **Enough history.** Default lookback is **30 days** (minimum 14), so it covers weekly cycles
   and month-end jobs. Ask whether there are known seasonal peaks (sales, launches, term
   starts) outside the window. Less than 14 days of data → verdict "insufficient data".
6. **Empty is not zero.** No memory metric usually means it isn't collected (e.g. EC2 has no
   memory metric in CloudWatch without an agent), not that memory is unused. Say "not
   measured", and don't downsize on CPU alone when memory is unknown.
7. **Every dimension counts.** A size change changes CPU, memory, network, storage throughput,
   and sometimes connection limits together. Check that the recommended type fits **all** of
   them, not just the one that looked idle.
8. **Ask only when blocked.** Use sensible defaults (below), state them, and proceed.

## Step 0 — Discover the toolbox

```bash
# Optional discovery for a compatible shell; API connectors and offline files also work.
for c in aws gcloud az kubectl oc tsh terraform terragrunt pulumi pup datadog-ci promtool \
         infracost jq gh glab; do
  command -v "$c" >/dev/null && echo "HAVE $c"
done
```

- **Connectors / MCP servers:** inspect the host's tool catalog or discovery interface for
  relevant inventory, metrics, and pricing sources. Confirm access with a read-only status
  call or query. Use the host's documented login flow if authentication is required.
- **Auth and scope:** confirm identity before querying. `aws sts get-caller-identity`,
  `gcloud config list`, `az account show`, `kubectl config current-context`. Access wrappers
  (e.g. `tsh aws --app <app>`, `tsh kubectl`, `aws-vault exec`) replace the plain command; use
  whichever the environment requires.
- **IaC:** check the current repo and the user's code directory for Terraform/Pulumi/Helm/CDK
  that define the resources (`grep -rl "instance_class\|instance_type\|node_type\|machine_type"`).
  This tells you where a change would be made and what's intended (HA, replicas).

Map what you found to capabilities:

| Capability | Examples |
|---|---|
| Inventory | `aws ec2/rds/elasticache describe-*`, `gcloud compute instances list`, `az vm list`, IaC |
| Utilization metrics | CloudWatch, Datadog, Prometheus, Azure Monitor, Cloud Monitoring |
| Type specs | Provider APIs/catalogs, retrieved provider documentation, or verified hardware/VM quotas; e.g. `aws ec2 describe-instance-types`, `gcloud compute machine-types list` |
| Prices | `aws pricing get-products`, `infracost`, provider pricing docs from the user |
| Native recommender | AWS Compute Optimizer, GCP Recommender, Azure Advisor |
| Commitments | RI / Savings Plans / CUD inventory (`aws ce`, `aws rds describe-reserved-db-instances`) |

Show the user a one-line summary of what's available and what's missing. If neither a live
metrics source nor a supplied historical export is available, stop and say so: right-sizing
without utilization data is guessing.

## Step 1 — Scope and inventory

### 1a. What's in scope

From the prompt and verified context, resolve resource kinds, account/project/subscription
or on-premises inventory, region(s), environment, and filters (tags, name prefix, a service).
Do not default to production, the active CLI account, a default region, or every resource
kind. Ask if the target is ambiguous; otherwise stay within the requested scope and state
the scope and coverage limits before querying.

If the user named one resource ("is `orders-db` oversized"), resolve it to its ID and skip
broad inventory.

### 1b. Inventory

For each resource, record: ID/name, kind, current type/size, region/AZ, key tags (service,
env, owner), and HA topology. Examples (AWS):

```bash
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].{id:InstanceId,type:InstanceType,az:Placement.AvailabilityZone,asg:Tags[?Key==`aws:autoscaling:groupName`]|[0].Value,name:Tags[?Key==`Name`]|[0].Value}'
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[].{name:AutoScalingGroupName,min:MinSize,max:MaxSize,desired:DesiredCapacity,lt:MixedInstancesPolicy||LaunchTemplate}'
aws rds describe-db-instances \
  --query 'DBInstances[].{id:DBInstanceIdentifier,class:DBInstanceClass,engine:Engine,ver:EngineVersion,multiAZ:MultiAZ,cluster:DBClusterIdentifier,replicaOf:ReadReplicaSourceDBInstanceIdentifier,storage:StorageType,iops:Iops}'
aws elasticache describe-replication-groups \
  --query 'ReplicationGroups[].{id:ReplicationGroupId,node:CacheNodeType,members:length(MemberClusters),clusterMode:ClusterEnabled}'
```

Kubernetes: list nodes with their instance type label (`node.kubernetes.io/instance-type`)
and nodepool/nodegroup, plus workloads with their CPU/memory requests and limits.

**Group, then sample.** Instances in the same ASG/nodegroup/replica set are sized together.
Analyze the group, not each member. With large fleets, rank by cost (type price × count)
and analyze the top items first. Say how many you skipped.

## Step 2 — Measure utilization

Get, per resource or group, **p95, p99, and max** over the lookback (and the mean, only for
context). Use a period that keeps peaks visible: 5 minutes or less. Hourly averages flatten
spikes.

Check each source's units, meaning, sample interval, and retention. Normalize cores vs
percent, bytes vs percentages, and free/available vs used memory before comparison.
Do not average percentiles or treat the percentile of hourly averages as a raw-sample
percentile. State the aggregation and effective coverage; use a shorter documented window
or mark insufficient data when the required resolution is unavailable. Cache, buffer, and
reclaimable-memory semantics vary by OS and service. Metric names below are candidates,
not portable definitions.

| Kind | Signals (candidates — verify they exist) | Binding-constraint hints |
|---|---|---|
| VM / EC2 | CPU %; memory % (agent: CWAgent `mem_used_percent`, Datadog `system.mem.pct_usable`, node_exporter); network in/out vs the type's bandwidth; EBS IOPS/throughput vs the type's EBS limit | T-family: `CPUCreditBalance` / `CPUSurplusCreditsCharged`. A credit balance near zero means it's under-provisioned even if CPU looks "fine" |
| RDS / Aurora / Cloud SQL | `CPUUtilization`, `FreeableMemory`, `SwapUsage`, `DatabaseConnections`, `ReadIOPS`/`WriteIOPS`, `ReadLatency`/`WriteLatency`, `DiskQueueDepth`, `NetworkThroughput`; Aurora: `BufferCacheHitRatio` | Low freeable memory or swap → memory-bound. Falling cache-hit ratio → the working set doesn't fit. Connections near `max_connections` |
| ElastiCache / Redis / Memcached | `EngineCPUUtilization` (Redis is mostly single-threaded; use this, not host CPU), `DatabaseMemoryUsagePercentage`, `Evictions`, `CurrConnections`, `NetworkBytesIn/Out`, `ReplicationLag` | Evictions > 0 with an eviction policy that matters → under-provisioned on memory |
| Search / streaming (OpenSearch, MSK/Kafka) | CPU, JVM memory pressure, disk used %, request/produce latency, partition/shard counts | Disk and JVM heap bind before CPU |
| Kubernetes nodes | Allocatable vs requested (bin-packing), actual usage vs allocatable, pending pods, node count over time | Low *requested* % → fragmentation or a too-large type. Low *usage* but high *requested* → the pods are over-requested (below) |
| Kubernetes workloads | Usage vs requests and limits: CPU (`container_cpu_usage_seconds_total` rate / Datadog `kubernetes.cpu.usage.total`), memory working set, CPU throttling, OOMKills, HPA replica range | Requests are what you pay for on nodes. Memory: size to peak working set. CPU: size to p95, and watch throttling |

`kubectl top` is a single point in time. Use it only to sanity-check, never for a verdict.

Check whether a **native recommender** has an opinion and pull it in as an input:

```bash
aws compute-optimizer get-enrollment-status
aws compute-optimizer get-ec2-instance-recommendations --instance-arns <arn>   # also: get-auto-scaling-group-recommendations, get-ebs-volume-recommendations, get-rds-database-recommendations
gcloud recommender recommendations list --project <p> --location <zone> --recommender google.compute.instance.MachineTypeRecommender
az advisor recommendation list --category Cost
```

Recommenders are evidence, not the answer. Compute Optimizer, for example, doesn't see EC2
memory without an agent and doesn't know about your seasonal peaks. Agree or disagree with it,
and say why.

## Step 3 — Classify

Default thresholds (state them, and let the user override):

| Verdict | Condition (over the full lookback) |
|---|---|
| **Insufficient data** | < 14 days of data, or the binding dimension (usually memory) not measured |
| **Under-provisioned candidate** | p95 CPU > 75%, peak memory > 85%, or evidence of saturation such as CPU-credit exhaustion, OOMKills, harmful eviction, sustained throttling, or growing disk latency/queue |
| **Over-provisioned candidate** | p99 CPU < 30%, peak memory < 50%, and no observed saturation; a smaller candidate still requires the checks in Step 4 |
| **Right-sized within observed scope** | Adequate evidence, no verified capacity bottleneck, and no smaller candidate passes all required checks |

These are starting heuristics, not universal policy. Interpret swap, evictions, memory
occupancy, connection limits, and throttling against the workload's behavior and objectives.
Check data sufficiency first. A directly observed OOM or other saturation event may still
be reported with limited history, but does not justify a confident sizing recommendation.

Then identify the **binding dimension**: whichever is closest to its limit. That decides the
family:

- CPU binds, memory idle → compute-optimized (e.g. AWS `c`)
- Memory binds, CPU idle → memory-optimized (`r`, `x`)
- Both moderate → general purpose (`m`)
- Spiky CPU with low baseline → burstable (`t`) *only* if the credit math works for the observed
  pattern. Avoid for databases and latency-sensitive services.

## Step 4 — Pick a candidate and verify it

For each candidate with sufficient evidence, evaluate **one recommended type or capacity
change and, at most, one alternative**. Do not recommend a size for an insufficient-data
resource. If no candidate passes the constraints, report that explicitly.

1. **Size.** For comparable, capacity-proportional dimensions, estimate
   `projected_utilization = observed_utilization × current_capacity / candidate_capacity`.
   Keep the statistic explicit: a projected p99 must use observed p99, not peak or mean.
   Default to at least **30% headroom** on the binding dimension at p99 (projected p99 ≤ 70%),
   then check observed peaks, failover, and burst behavior separately. Scale only dimensions
   where the approximation is valid; memory caches, IOPS limits, and network baselines may
   not scale linearly. Validate those from service behavior and candidate specifications.
   For a CPU-generation change, treat CPU as roughly comparable and say that's an
   approximation; don't claim a specific performance gain unless the user provided a benchmark.
2. **Generation and architecture.** Prefer the current generation of the same family when
   available. ARM (e.g. Graviton) is a candidate only if you can confirm compatibility:
   multi-arch images, a supported DB engine/version, and no x86-only binaries. Otherwise list
   it as "possible, needs compatibility check".
3. **Verify the type exists where it's needed:**
   ```bash
   aws ec2 describe-instance-types --instance-types <t1> <t2> \
     --query 'InstanceTypes[].{t:InstanceType,vcpu:VCpuInfo.DefaultVCpus,memMiB:MemoryInfo.SizeInMiB,net:NetworkInfo.NetworkPerformance,arch:ProcessorInfo.SupportedArchitectures,burst:BurstablePerformanceSupported}'
   aws ec2 describe-instance-type-offerings --location-type availability-zone \
     --filters Name=instance-type,Values=<t> --region <r>
   aws rds describe-orderable-db-instance-options --engine <engine> --engine-version <ver> \
     --db-instance-class <class> --query 'OrderableDBInstanceOptions[].{az:AvailabilityZones[].Name,multiAZ:MultiAZCapable,storage:StorageType}'
   ```
   An offerings/orderability API establishes availability, not necessarily memory, vCPU,
   I/O, or network specifications. Retrieve missing specs from the relevant provider's
   catalog or documentation; do not infer managed-service specs from similarly named VMs.
4. **Price both** (current and candidate) for the same region, OS/engine, and tenancy:
   ```bash
   aws pricing get-products --region <pricing-api-region> --service-code AmazonEC2 --filters \
     Type=TERM_MATCH,Field=instanceType,Value=<t> Type=TERM_MATCH,Field=regionCode,Value=<r> \
     Type=TERM_MATCH,Field=operatingSystem,Value=Linux Type=TERM_MATCH,Field=tenancy,Value=Shared \
     Type=TERM_MATCH,Field=preInstalledSw,Value=NA Type=TERM_MATCH,Field=capacitystatus,Value=Used
   # RDS: --service-code AmazonRDS with instanceType, databaseEngine, deploymentOption (Single-AZ/Multi-AZ)
   # ElastiCache: --service-code AmazonElastiCache with instanceType, cacheEngine
   ```
   For APIs with regional endpoints, discover a supported pricing API region for the
   account's partition; it may differ from the resource region used in product filters.
   For hourly billing, a 730-hour monthly equivalent is an explicit comparison assumption,
   not an invoice forecast. Match currency, billing unit, OS/engine, licensing, tenancy,
   and region, and distinguish list-price deltas from realizable savings. Use the actual
   billing model for non-hourly or on-premises resources. If `infracost` and IaC are available,
   an existing estimate may provide supporting pricing evidence without editing the repo.

### Constraints to check before recommending

- **Commitments.** Reserved Instances / Savings Plans / CUDs. A downsize inside an RI's family
  may still be covered (size flexibility); a family change may strand the commitment. Check
  coverage (`aws ce get-reservation-coverage`, `get-savings-plans-coverage`) and say so.
- **HA and failover.** A Multi-AZ standby or failover replica must handle the full load alone.
  Read replicas that can be promoted should match the writer. DR copies are sized for failover
  traffic, not their current idle traffic. Don't call them oversized for being idle.
- **Limits tied to size.** DB `max_connections` often derives from instance memory (check the
  parameter group). Smaller sizes have lower network and EBS bandwidth baselines. Cache memory
  must fit the dataset plus reserved memory and replication buffers.
- **Autoscaled groups.** For ASGs, nodegroups, and HPA workloads, the right lever may be
  min/max/target rather than type. Recommend whichever fixes the binding dimension with less
  risk.
- **Downtime.** Resizing a single-AZ DB or a non-cluster-mode cache usually means an outage or
  failover. Note the expected impact; don't decide the change window for the user.

## Parallelize

If delegation is supported, collect independent resource kinds (VMs, databases, caches,
Kubernetes) in parallel using the host's configured model or a suitable available model.
Otherwise collect them sequentially. Keep sources inaccessible to workers in the main
session and do not pass credentials to workers. Give each worker the exact scope, lookback,
tool/wrapper to use, and the signal list for its kind. Have them return
**raw numbers** (p95/p99/max per signal, current type, data coverage in days) with no
verdicts. The main agent classifies, picks candidates, and prices them, so thresholds and
judgment stay consistent.

## Step 5 — Report

```
## Scope
<accounts/projects, regions, env, kinds, lookback (dates, UTC), thresholds used, tools used, assumptions>

## Summary
<N resources analyzed: X over, Y under, Z right-sized, W insufficient data. Net on-demand delta if all applied: $/mo (or "prices not retrieved")>

## Recommendations
| Resource | Kind | Current | p95/p99/max CPU | Peak mem | Other binding signal | Verdict | Recommended (alt) | Projected p99 on binding dim | $/mo now → new | Risks / notes | Where to change |
|---|---|---|---|---|---|---|---|---|---|---|---|
| <resource> | <kind/topology> | <verified capacity> | <retrieved values> | <retrieved value> | <signal and limit> | <verdict> | <verified candidate or none> | <calculation and assumptions> | <retrieved prices or unavailable> | <verified constraints and unknowns> | <discovered IaC path or unknown> |

## Under-provisioned (act first)
<the under-provisioned items with the evidence. They are reliability risks, not just cost>

## Native recommender comparison
<where you agree/disagree with Compute Optimizer / Recommender / Advisor and why>

## Not checked / gaps
<missing metrics (e.g. EC2 memory), resources skipped, prices not retrieved, commitments not checked, short history>
```

Rank recommendations by **risk first, then savings**: under-provisioned items, then the largest
safe savings, then marginal ones. Mark anything where the binding dimension wasn't measured as
"insufficient data", even if CPU looks idle.

## Install

From the root of this repository:

For Claude Code on a POSIX-like shell:

```bash
mkdir -p "$HOME/.claude/skills"
dest="$HOME/.claude/skills/right-size"
if [ -e "$dest" ] || [ -L "$dest" ]; then
  printf 'Already exists: %s\n' "$dest"
else
  ln -s "$PWD/right-size" "$dest"
fi
```

Alternatively, copy this directory into a project's `.claude/skills/` directory.
Then invoke with `/right-size <resource | service | "all prod RDS" | account/region>`.
