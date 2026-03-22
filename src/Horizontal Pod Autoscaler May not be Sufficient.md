# Horizontal Autoscaler May not be Sufficient.


Traffic spikes at 9 AM every Monday.  Your application running on kubernetes is supposed to handle the spike, but by the time it handles the users have already started noticing the latency.

This is a problem with a slow reactor.  

Kubernetes has a concept of Horizontal Pod Autoscaler, which scales up the deploy, based on a certain utilization metric.  By the time HPA notices, scales up, the scheduler places pods, images pull, containers start, and readiness probes pass, it's been 2-4 minutes. Users already saw degraded latency. 

Primary reason for this is, Horizontal Pod Autoscaler is purely reactive, it scales on symptoms (high CPU) not causes (incoming traffic).

Here's what Horizontal Pod Autoscaler looks like if you squint, it's just a control loop.



![](Pasted%20image%2020260321150332.png)


At this level, the story is simple. Kubernetes implements horizontal pod autoscaling as a control loop that runs intermittently (it is not a continuous process)

The default interval is 15 seconds, controlled by `--horizontal-pod-autoscaler-sync-period`. Every tick, it asks "what are the current metrics?", computes the desired replica count using a simple ratio formula, and writes the result to the Deployment's scale subresource.

**The core formula** is beautifully simple:

![](Pasted%20image%2020260321150626.png)


When you start with 3 pods, If current CPU is 200m and target is 100m then `ceil(3 × 200/100) = 6`, the deployment will be scaled out to 6 replicas. If current is 50m → `ceil(3 × 50/100) = 2`, the deployment will be scaled down to 2 replicas..


Having a metrics-server is essential for the kubernetes horizontal pod autoscaler to work. Otherwise, you will see an error -"Metrics API not available" and HPA resource won' be able to do it's work.



![](Pasted%20image%2020260321151125.png)


This is where the real complexity lives. There are **three separate metrics APIs** that HPA can query:

**`metrics.k8s.io`** — Resource metrics (CPU, memory). Backed by metrics-server, which scrapes kubelet's Summary API every ~60 seconds. kubelet in turn gets container stats from cAdvisor (built-in) or the CRI. This is the simplest path but also the most limited.

**`custom.metrics.k8s.io`** — Custom metrics from within the cluster. Typically backed by Prometheus Adapter, which translates Prometheus queries into the Kubernetes Custom Metrics API. This is where you'd serve `requests_per_second`, `queue_depth`, or — something relevant to your GPU work — `DCGM_FI_DEV_GPU_UTIL` via the DCGM exporter → Prometheus → Prometheus Adapter pipeline.

**`external.metrics.k8s.io`** — Metrics from outside the cluster. Cloud provider queue depths, SaaS API call counts, etc.

The common use for HorizontalPodAutoscaler is to configure it to fetch metrics from aggregated APIs. [kubernetes](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/) The API server's aggregation layer routes requests to the appropriate backend (metrics-server, Prometheus Adapter, etc.).

The **total latency** in this pipeline matters enormously. From the source code perspective (in `pkg/controller/podautoscaler/horizontal.go`), the reconcile loop looks roughly like:

```go
// Simplified from kubernetes/pkg/controller/podautoscaler/horizontal.go
func (a *HorizontalController) reconcileAutoscaler(ctx context.Context, hpa *autoscalingv2.HorizontalPodAutoscaler) error {
    // 1. Get the scale subresource for the target
    scale, err := a.scaleNamespacer.Scales(hpa.Namespace).Get(ctx, targetGR, hpa.Spec.ScaleTargetRef.Name, ...)
    
    // 2. Calculate desired replicas based on metrics
    desiredReplicas, condition := a.computeReplicasForMetrics(ctx, hpa, scale, hpa.Spec.Metrics)
    
    // 3. Apply stabilization window — pick highest recommendation in the window
    desiredReplicas = a.stabilizeRecommendation(key, desiredReplicas)
    
    // 4. Clamp to [minReplicas, maxReplicas]
    desiredReplicas = max(minReplicas, min(maxReplicas, desiredReplicas))
    
    // 5. Rescale if different from current
    if desiredReplicas != scale.Spec.Replicas {
        scale.Spec.Replicas = desiredReplicas
        _, err = a.scaleNamespacer.Scales(hpa.Namespace).Update(ctx, targetGR, scale, ...)
    }
}
```


_The HPA controller wakes up on its next 15-second tick. It queries metrics-server. It sees the spike. It calculates: `ceil(3 × 92/50) = 6` — need 6 pods. It writes the new replica count._

_The scheduler finds nodes, the kubelet pulls the image (already cached, thankfully), the container starts, the readiness probe begins its checks..._

_By the time the 6th pod passes its readiness probe and enters the Service's endpoint pool, 2 minutes and 5 seconds have passed since the spike began. During that window, your 3 original pods absorbed all the extra load. Users saw degraded latency. Some saw errors._

_The HPA did exactly what it was supposed to do. And it was still too late._


![](Pasted%20image%2020260321154156.png)

The brownout window timeline breaks down roughly as: T+30-45s the HPA controller polls the Metrics Server, sees the problem, and decides to scale; T+45-90s the scheduler finds nodes and the kubelets begin pulling container images; T+90-180s and beyond the application itself starts up. ScaleOps For JVM-heavy services with cold image pulls on new nodes, you're looking at 3+ minutes easily.
The fundamental insight is this: HPA scales based on observed resource utilization, not anticipated demand. ScaleOps By the time CPU rises enough to trigger scaling, the traffic spike has already impacted user experience. Every Monday morning is treated as a novel event, even when the pattern has repeated for years.

The Metrics Pipeline — Why the Data Arrives Late

The delay isn't just "the HPA controller is slow." The data itself is stale by the time HPA sees it. Let me trace the journey of a single CPU measurement.

image


Here's the critical detail most people miss. The Linux kernel exposes container CPU time as a monotonically increasing counter (container_cpu_usage_seconds_total in cgroups v2). To get a "CPU usage percentage," something has to compute a rate — the change in that counter over a time window, divided by the elapsed time. This means:
The CPU percentage HPA sees is always an average over some lookback window, not a snapshot. The nature of the metric measurement smooths away the disaster and renders the autoscaler blind to the real user pain. ScaleOps If you had a 5-second spike to 100% CPU followed by 10 seconds at 30%, the averaged value might show a mild 53% — well below your 60% threshold. The spike is invisible.
From the Kubernetes source code (pkg/controller/podautoscaler/horizontal.go), the controller in reconcileAutoscaler() fetches metrics, then runs through this decision chain:


```go
// Simplified from kubernetes/kubernetes source
// Step 1: Fetch metrics for all pods
metrics, timestamp, err := a.metricsClient.GetResourceMetric(...)

// Step 2: Compute ratio
usageRatio, _, _, err := metricsclient.GetResourceUtilizationRatio(metrics, requests, targetUtilization)

// Step 3: Tolerance check — THIS is where small spikes get swallowed
if math.Abs(1.0-usageRatio) <= tolerance {   // default tolerance = 0.1 (10%)
    return currentReplicas, timestamp, nil    // DO NOTHING
}

// Step 4: Calculate desired
desiredReplicas := int32(math.Ceil(usageRatio * float64(readyPodCount)))
```


That tolerance check on line 3 is another hidden delay. If your target is 50% and current is 54%, the ratio is 1.08 — within the 10% tolerance band. HPA does nothing. The CPU has to reach ~56% (ratio > 1.1) before HPA even considers scaling.


That ~30 second metrics pipeline delay is a hard floor you can't eliminate with HPA alone.


Let's trace this through the actual Kubernetes source code. The HPA controller lives at pkg/controller/podautoscaler/horizontal.go. Here's the simplified reconcile loop:


```

go// pkg/controller/podautoscaler/horizontal.go (simplified)
func (a *HorizontalController) reconcileAutoscaler(ctx context.Context, 
    hpa *autoscalingv2.HorizontalPodAutoscaler) error {
    
    // 1. Get current scale
    scale, targetGR, err := a.scaleForResourceMappings(ctx, hpa.Namespace, 
        hpa.Spec.ScaleTargetRef, mappings)
    currentReplicas := scale.Status.Replicas
    
    // 2. Compute desired replicas from ALL metrics (takes the max)
    metricDesiredReplicas, metricName, metricStatuses, metricTimestamp, err := 
        a.computeReplicasForMetrics(ctx, hpa, scale, hpa.Spec.Metrics)
    // ← THIS is where the 15-75s stale metric is consumed
    
    // 3. Apply stabilization — look back over the window
    stabilizedRecommendation := a.stabilizeRecommendation(key, metricDesiredReplicas)
    // ← For scale-down, looks at last 5 min of recommendations
    
    // 4. Apply behavior rate limits  
    desiredReplicas := a.convertDesiredReplicasWithBehaviorRate(...)
    // ← "Only add 100% of current per 60s" type limits
    
    // 5. Clamp to [min, max]
    desiredReplicas = max(*hpa.Spec.MinReplicas, min(hpa.Spec.MaxReplicas, desiredReplicas))
    
    // 6. Update the scale subresource
    if desiredReplicas != currentReplicas {
        scale.Spec.Replicas = desiredReplicas
        _, err = a.scaleNamespacer.Scales(hpa.Namespace).Update(ctx, targetGR, scale, ...)
        // → This triggers the Deployment controller → ReplicaSet → new pods
    }
}
```

The key function is computeReplicasForMetrics. For each metric spec, it calls computeReplicasForMetric, which for resource metrics ends up in replica_calculator.go:

```
go// pkg/controller/podautoscaler/replica_calculator.go (simplified)
func (c *ReplicaCalculator) GetResourceReplicas(...) (int32, ...) {
    // Fetch metrics from metrics.k8s.io 
    metrics, timestamp, err := c.metricsClient.GetResourceMetric(resource, namespace, selector, container)
    
    // Remove unready pods, missing pods, ignored pods
    readyPodCount, unreadyPods, missingPods, ignoredPods := groupPods(podList, metrics, ...)
    
    // Compute utilization ratio
    requests, err := calculatePodRequests(podList, container, resource)
    usageRatio, utilization, rawUsage, err := 
        metricsclient.GetResourceUtilizationRatio(metrics, requests, targetUtilization)
    
    // THE TOLERANCE CHECK — this is where small spikes get ignored
    if math.Abs(1.0 - usageRatio) <= c.tolerance {
        return currentReplicas, utilization, rawUsage, timestamp, nil
    }
    
    // Adjust for missing/unready pods
    // ... (complex logic for edge cases)
    
    return int32(math.Ceil(usageRatio * float64(readyPodCount))), ...
}
```


The Fixes: From Reactive to Proactive

Now that we understand the anatomy of the delay, here are the production strategies from simplest to most sophisticated:


Fix 1: The N+2 Buffer (5 minutes, zero risk)
Run more pods than your baseline requires. If you need 3 for normal traffic, run 5. Two extra pods absorb the spike while HPA catches up.

```
yamlspec:
  minReplicas: 5    # Not 3. 2 buffer pods = availability insurance
  maxReplicas: 20

```

Fix 2: Tighten the metrics pipeline (medium effort)

```
bash# Reduce HPA sync period from 15s to 10s (on kube-controller-manager)
--horizontal-pod-autoscaler-sync-period=10s
```


```
# Reduce metrics-server scrape interval
# In the metrics-server Deployment args:
- --metric-resolution=30s    # default is 60s
```

Fix 3: Aggressive scale-up behavior (the right YAML)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ml-inference-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ml-inference
  minReplicas: 5           # N+2 buffer
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0     # React immediately to spikes
      policies:
      - type: Percent
        value: 200                      # Triple capacity in one shot
        periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300   # But be slow to scale down
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60

```


Fix 4: Scale on leading indicators, not lagging ones

CPU is a lagging indicator — it rises after traffic hits. Scale instead on requests_per_second (a concurrent indicator) or queue depth (a leading indicator):

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second      # From Prometheus Adapter
    target:
      type: AverageValue
      averageValue: "100"                 # Scale when > 100 RPS per pod
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70              # CPU as safety net only

```


Fix 5: Predictive pre-scaling with CronJobs

For the Monday morning problem specifically — you know when it's coming:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: monday-prescale
spec:
  schedule: "50 8 * * 1"     # 8:50 AM every Monday, 10 min before the spike
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: prescaler
            image: bitnami/kubectl
            command:
            - /bin/sh
            - -c
            - kubectl scale deployment ml-inference --replicas=10
          restartPolicy: OnFailure

```

Predictive Scaling by employing other technologies.

Uber has an entire stack of predictive scaling pipeline, and Uber warms it's instances before the traffic peaks at regular events. Like when people are most likely to summon a cab or a order food. This predictive scaling helps to maintain the low latency output in very large infra.

