Network partitions are not theoretical. A switch goes down, a power event hits a data center, and suddenly one availability zone goes dark. The question is never if this happens — it's whether your services notice before your customers do.

Here's what I learned building a continuous AZ-level monitoring system for Kubernetes networking at scale.

  ---
  What Actually Breaks During an AZ Partition

  When an AZ loses connectivity, Kubernetes doesn't fail cleanly. It fails ambiguously. Three distinct layers can degrade independently:

  1. The data plane (pod networking)
  VPC CNI stops allocating IPs from the affected subnet. Pods on impacted nodes can't be scheduled. Existing pods may lose east-west connectivity if their peer pods reside in the partitioned AZ.

  2. The service proxy layer (kube-proxy)
  kube-proxy maintains iptables rules for Service routing. If the API server becomes unreachable from nodes in the partitioned AZ, kube-proxy stops receiving Endpoints updates. Traffic continues routing to stale endpoints — including ones that are now unreachable.

  3. DNS (CoreDNS)
  CoreDNS pods themselves may land in the partitioned AZ. DNS resolution silently fails or times out, causing cascading failures that look like application bugs.

  ---
  The Kubernetes Resilience Toolkit

  Topology-Aware Routing

  Use topologyKeys in Services or the newer trafficDistribution: PreferClose to bias traffic toward same-zone endpoints. This limits blast radius — a partition in us-east-1d stops poisoning traffic flows that don't need to cross zones.

  apiVersion: v1
  kind: Service
  spec:
    trafficDistribution: PreferClose

  Anti-Affinity for Critical Components

  Spread your critical pods — especially CoreDNS — across AZs using podAntiAffinity. A single-AZ CoreDNS outage should not take down DNS for the entire cluster.

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: topology.kubernetes.io/zone

  EndpointSlice Health Signals

  Enable EndpointSlice (default in modern Kubernetes) and configure proper readiness probes. When a node in the partitioned AZ becomes unreachable, the node controller marks pods NotReady and they are removed from endpoint slices — but only after the node-monitor-grace-period (default: 40s). Size this window consciously.

  PodDisruptionBudgets Aren't Enough

  PDBs protect against voluntary disruptions. A network partition is involuntary. You need redundancy built into your replica count and zone spread — not just a minAvailable safety net.

  ---
  The Hard Part: Knowing You're Partitioned

  The biggest operational gap is detection. During a real AZ outage, you often fall back to hypotheticals: "If a customer was using X, they would have seen Y." That's not good enough for incident response.

  The solution is continuous synthetic testing that exercises the full networking stack from within the cluster:

  - Pod presence per AZ — verify nodes and schedulable pods exist in every zone
  - Pod-to-pod TCP connectivity across AZ boundaries — catches data plane partitions
  - Internal API server reachability via kube-proxy — catches control plane degradation
  - External NLB endpoint reachability — catches ingress path failures
  - DNS resolution via CoreDNS — catches service discovery failures

  Run these every 2 minutes, across every AZ, in parallel. Alert on no signal (not just failures) — a silent test runner is as bad as a failing one.

  ---
  Eliminating False Signals

  When your monitoring itself has dependencies that fail during a partition, you get noise when you need signal. Common mistakes:

  - Calling cloud APIs (e.g., DescribeSubnets) from within the test — the API itself may be impacted
  - Dynamic pod scheduling at test time — if the scheduler is degraded, your test never starts
  - Image pulls during incident — the registry endpoint may be unavailable

  Pre-deploy everything. Use DaemonSets so pods are already running on every node before the incident. Cache images. Eliminate runtime dependencies on AWS/GCP APIs. Your monitoring should be a passive observer, not an active participant in the infrastructure it's watching.

  ---
  What Good Looks Like

  When an event occurs, you should be able to answer within minutes:

  - Which AZs are affected?
  - Is pod-to-pod connectivity impacted, or only control plane?
  - Are external-facing endpoints still reachable?
  - Which specific networking components (CNI, kube-proxy, CoreDNS) show failures?

  With continuous synthetic tests publishing to a metrics dashboard, this becomes a lookup — not a reconstruction from customer tickets and log archaeology.

  ---
  Key Takeaways

  1. AZ partitions degrade multiple independent layers — don't assume one failure mode.
  2. Topology-aware routing limits blast radius — keep traffic local when possible.
  3. Spread everything — CoreDNS, critical services, monitoring agents across zones.
  4. Synthetic testing beats reactive alerting — know your state before your customers do.
  5. Your monitoring must survive the incident it's monitoring — eliminate infrastructure dependencies from the test path.

  Network partitions will happen. The difference between a measured, confident incident response and a chaotic one comes down to whether you built the observability before the pager went off.


![](Pasted%20image%2020260321221203.png)

![](Pasted%20image%2020260321221236.png)