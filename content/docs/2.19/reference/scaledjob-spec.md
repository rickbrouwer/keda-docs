+++
title = "ScaledJob specification"
weight = 4000
+++

## Overview

This specification describes the `ScaledJob` custom resource definition that defines how KEDA scales jobs based on event-driven triggers. A ScaledJob monitors external metrics and automatically creates Kubernetes Jobs when work needs to be processed.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: {scaled-job-name}
  labels:
    my-label: {my-label-value}                # Optional. ScaledJob labels are applied to child Jobs
  annotations:
    autoscaling.keda.sh/paused: true          # Optional. Use to pause autoscaling of Jobs
    my-annotation: {my-annotation-value}      # Optional. ScaledJob annotations are applied to child Jobs
spec:
  jobTargetRef:
    parallelism: 1                            # [max number of desired pods](https://kubernetes.io/docs/concepts/workloads/controllers/job/#controlling-parallelism)
    completions: 1                            # [desired number of successfully finished pods](https://kubernetes.io/docs/concepts/workloads/controllers/job/#controlling-parallelism)
    activeDeadlineSeconds: 600                #  Specifies the duration in seconds relative to the startTime that the job may be active before the system tries to terminate it; value must be positive integer
    backoffLimit: 6                           # Specifies the number of retries before marking this job failed. Defaults to 6
    template:
      # describes the job's [pod template](https://kubernetes.io/docs/concepts/workloads/controllers/job)
  pollingInterval: 30                         # Optional. Default: 30 seconds
  successfulJobsHistoryLimit: 5               # Optional. Default: 100. How many completed jobs should be kept.
  failedJobsHistoryLimit: 5                   # Optional. Default: 100. How many failed jobs should be kept.
  envSourceContainerName: {container-name}    # Optional. Default: .spec.JobTargetRef.template.spec.containers[0]
  minReplicaCount: 10                         # Optional. Default: 0
  maxReplicaCount: 100                        # Optional. Default: 100
  rolloutStrategy: gradual                    # Deprecated: Use rollout.strategy instead (see below).
  rollout:
    strategy: gradual                         # Optional. Default: default. Which Rollout Strategy KEDA will use.
    propagationPolicy: foreground             # Optional. Default: background. Kubernetes propagation policy for cleaning up existing jobs during rollout.
  scalingStrategy:
    strategy: "custom"                        # Optional. Default: default. Which Scaling Strategy to use. 
    customScalingQueueLengthDeduction: 1      # Optional. A parameter to optimize custom ScalingStrategy.
    customScalingRunningJobPercentage: "0.5"  # Optional. A parameter to optimize custom ScalingStrategy.
    pendingPodConditions:                     # Optional. A parameter to calculate pending job count per the specified pod conditions
      - "Ready"
      - "PodScheduled"
      - "AnyOtherCustomPodCondition"
    multipleScalersCalculation : "max"        # Optional. Default: max. Specifies how to calculate the target metrics when multiple scalers are defined.
  triggers:
  # {list of triggers to create jobs}
```

You can find all supported triggers [here](../scalers).

---

## Job Target Reference

The `jobTargetRef` defines the template for Jobs that KEDA will create. This is a standard Kubernetes batch/v1 JobSpec - refer to the [Kubernetes Job documentation](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/job-v1/#JobSpec) for complete details.

### Key Configuration Options

**parallelism** - Controls how many pods can run simultaneously within a single Job. For example, setting this to 3 means each Job can process up to 3 work items concurrently.

**completions** - Defines how many successful pod completions are required before a Job is considered finished. Combined with parallelism, this determines the total work capacity of each Job.

**activeDeadlineSeconds** - Sets a maximum runtime for Jobs. If a Job exceeds this duration, Kubernetes will terminate it. This prevents runaway Jobs from consuming resources indefinitely.

**backoffLimit** - Determines how many times Kubernetes will retry a failed pod before marking the entire Job as failed. The default is 6 retries.

**template** - The pod template that defines what containers run in each Job, their resources, volumes, and other pod specifications. This field is required.

---

## Polling Interval

```yaml
pollingInterval: 30  # Optional. Default: 30 seconds
```

KEDA checks each trigger at regular intervals to determine if new Jobs need to be created. The polling interval defines how frequently these checks occur. A shorter interval means faster response to new work, but increases the load on your trigger sources (like message queues or databases). The default of 30 seconds provides a good balance for most workloads.

---

## Job History Limits

```yaml
successfulJobsHistoryLimit: 5  # Optional. Default: 100
failedJobsHistoryLimit: 5      # Optional. Default: 100
```

These settings control how many completed and failed Jobs KEDA retains for inspection and debugging. Keeping some job history is useful for troubleshooting and auditing, but too many old Jobs can clutter your cluster.

**successfulJobsHistoryLimit** - The number of successfully completed Jobs to keep. Once this limit is exceeded, KEDA deletes the oldest completed Jobs during the next cleanup cycle.

**failedJobsHistoryLimit** - The number of failed Jobs to retain. This helps you investigate what went wrong without keeping failed Jobs indefinitely.

Note that the actual number of Jobs may temporarily exceed these limits immediately after completion, but KEDA will clean them up during the next polling interval.

---

## Environment Source Container

```yaml
envSourceContainerName: {container-name}  # Optional
```

Some KEDA scalers need to read environment variables or secrets from your Job's containers to connect to trigger sources. This setting tells KEDA which container to inspect for these values.

If not specified, KEDA will use the first container in your Job template. Only configure this if you have multiple containers and need KEDA to read from a specific one.

---

## Replica Count Limits

### Minimum Replica Count

```yaml
minReplicaCount: 10  # Optional. Default: 0
```

The minimum number of Jobs that should always be running, even when there's no work in the queue. This is useful when you want to avoid the cold-start time of creating new Jobs.

For example, if you set `minReplicaCount` to 2, KEDA ensures at least 2 Jobs are always active. When new messages arrive, KEDA may create additional Jobs up to the `maxReplicaCount`, but will always maintain your minimum baseline.

If you set `minReplicaCount` higher than `maxReplicaCount`, KEDA will automatically cap it at the maximum value.

### Maximum Replica Count

```yaml
maxReplicaCount: 100  # Optional. Default: 100
```

The maximum number of Jobs that KEDA can create during a single polling interval. This acts as a safety limit to prevent runaway scaling that could overwhelm your cluster.

The actual number of Jobs created depends on several factors:
- The queue length reported by your scaler
- The target average value (how many items each Job should process)
- How many Jobs are already running
- Your configured scaling strategy

Example scenarios:

| Queue Length | Max Replica Count | Target Per Job | Running Jobs | Jobs Created |
|--------------|-------------------|----------------|--------------|--------------|
| 10           | 3                 | 1              | 0            | 3            |
| 10           | 3                 | 2              | 0            | 3            |
| 10           | 3                 | 1              | 1            | 2            |
| 10           | 100               | 1              | 0            | 10           |
| 4            | 3                 | 5              | 0            | 1            |

---

## Rollout Strategy

```yaml
rollout:
  strategy: gradual              # Optional. Default: default
  propagationPolicy: foreground  # Optional. Default: background
```

When you update a ScaledJob configuration, the rollout strategy determines how KEDA handles existing Jobs.

### Strategy Options

**default** - KEDA immediately terminates all existing Jobs when you update the ScaledJob. New Jobs are then created with the updated configuration. This ensures all Jobs use the latest settings, but may interrupt in-progress work.

**gradual** - Existing Jobs continue running with their original configuration. Only newly created Jobs use the updated settings. This prevents interruption but means old and new Job versions will coexist until the old Jobs complete.

### Propagation Policy

When using the `default` rollout strategy, the propagation policy controls how Kubernetes deletes existing Jobs:

**background** (default) - Kubernetes deletes Jobs immediately and cleans up their pods asynchronously. This is faster but pods may continue running briefly.

**foreground** - Kubernetes waits for all pods to terminate before marking the Job as deleted. This ensures complete cleanup but takes longer.

For more details, see the [Kubernetes cascading deletion documentation](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/#use-foreground-cascading-deletion).

---

## Scaling Strategy

The scaling strategy determines how KEDA calculates the number of Jobs to create based on queue metrics. Different strategies optimize for different use cases.

```yaml
scalingStrategy:
  strategy: "default"  # Optional. Default: default
```

### Understanding Scaling Metrics

Before diving into strategies, it's important to understand the key metrics KEDA uses:

**maxScale** - The theoretical maximum number of Jobs needed based on queue length and target average value. This is calculated by dividing the total queue length by how many items each Job should process, capped at your maxReplicaCount.

**runningJobCount** - Jobs that are currently active (not yet finished, whether running, pending, or failed).

**pendingJobCount** - Jobs that exist but whose pods haven't started processing work yet. By default, this counts Jobs whose pods aren't running or completed. You can customize this with `pendingPodConditions`.

### Strategy: default

The simplest strategy that works well for most scenarios. KEDA creates Jobs based on the actual queue length, accounting for already-running Jobs.

The number of Jobs created is determined by:
- Taking the queue length divided by target average value (how many items each Job should process)
- Subtracting the number of already running Jobs
- Capping at maxReplicaCount

This prevents over-provisioning by only creating Jobs when there's actual work in the queue that isn't already being handled.

### Strategy: custom

Provides fine-grained control over scaling decisions through two tunable parameters:

```yaml
customScalingQueueLengthDeduction: 1      # Optional
customScalingRunningJobPercentage: "0.5"  # Optional
```

**customScalingQueueLengthDeduction** - Reduces the calculated maxScale by a fixed amount. Use this if your scalers slightly overestimate queue length or to build in a buffer.

**customScalingRunningJobPercentage** - Applies a percentage reduction based on running Jobs. For example, "0.5" means each running Job reduces the allowed new Jobs by 50% of one Job. This gives you partial credit for running Jobs that may be about to finish.

Without these parameters configured, this strategy behaves like `default`.

### Strategy: accurate

Optimized for queue systems where the reported queue length doesn't include messages already being processed (locked/in-flight messages). Azure Storage Queue is a typical example.

This strategy accounts for pending Jobs more precisely:
- If maxScale plus running Jobs would exceed maxReplicaCount, it caps at the maximum
- Otherwise, it subtracts pending Jobs from maxScale to avoid over-provisioning

Use this when your scaler reports only waiting messages, not messages currently being processed by existing Jobs.

### Strategy: eager

Maximizes Job creation to use all available capacity up to maxReplicaCount. The key difference from other strategies is that eager always tries to create as many Jobs as allowed, regardless of the exact queue length calculation.

While the default strategy calculates precisely how many Jobs are needed based on queue length, eager fills all available slots (maxReplicaCount minus running and pending Jobs). This ensures work gets processed as fast as possible at the cost of potentially over-provisioning.

**Example scenario:**

Configuration: maxReplicaCount of 10, Jobs that run for 3 hours, pollingInterval of 10 seconds, target value of 1 message per Job.

You send 3 messages to RabbitMQ, wait longer than the polling interval, then send 3 more messages.

With `default` strategy:
- After first messages: 3 Jobs created (queue has 3 messages)
- After second poll (with 3 more messages): Queue now has 6 messages, but only 3 more Jobs created (total 6)
- The calculation is: (6 messages / 1 per job) - 3 running = 3 new Jobs

With `eager` strategy:
- After first messages: 3 Jobs created (queue has 3 messages)
- After second poll (with 3 more messages): Queue has 6 messages, and 6 more Jobs created (total 9)
- The strategy tries to fill capacity: min(10 maxReplica - 3 running, 6 needed) = 6 new Jobs, but returns maxReplicaCount to signal "create as many as possible"

The eager strategy is more aggressive in utilizing available capacity, which reduces queue wait time but may create more Jobs than strictly needed based on current queue length.

---

## Pending Pod Conditions

```yaml
scalingStrategy:
  pendingPodConditions:
    - "Ready"
    - "PodScheduled"
```

By default, KEDA treats a Job as "pending" if its pod hasn't started running or completed. This works fine for Jobs that start instantly, but causes problems when Jobs need initialization time.

### The startup time problem

Consider a Job that needs 30 seconds to set up database connections and load caches. Here's what happens:

- KEDA sees the pod running and marks the Job as active
- The Job can't process messages yet because it's still initializing
- KEDA thinks there's enough capacity and doesn't create more Jobs
- Messages sit in the queue waiting for initialization to finish

You end up under-scaled because KEDA has a false picture of your available capacity.

### How pending pod conditions help

The `pendingPodConditions` setting lets you specify which Kubernetes pod conditions must be met before KEDA counts a Job as active. The "Ready" condition is usually what you want.

### Connecting this to readiness probes

The "Ready" pod condition is controlled by your readinessProbe configuration. Without a readinessProbe, Kubernetes marks a pod as ready the moment it starts. With a readinessProbe, the pod only becomes ready after your probe succeeds.

Set up a readinessProbe that checks whether your application has finished initializing:

```yaml
jobTargetRef:
  template:
    spec:
      containers:
      - name: worker
        image: my-worker:latest
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

The `/ready` endpoint in your application should return success only after database connections are up, caches are loaded, and everything is actually ready to process work.

Then configure KEDA to wait for this readiness signal:

```yaml
scalingStrategy:
  strategy: accurate
  pendingPodConditions:
    - "Ready"
```

This combination works well with the `accurate` strategy since both focus on getting an accurate picture of available capacity.

### Other pod conditions

**PodScheduled** is useful when pods might wait for node resources. Requiring both "Ready" and "PodScheduled" means KEDA only counts Jobs that have been scheduled *and* finished initializing.

You can also reference custom pod conditions if you're using operators or admission controllers that set them.

### When to configure this

This is worth setting up if your Jobs have meaningful startup time—things like database connections, cache warming, or model loading. It's particularly important with the `accurate` scaling strategy and queue systems that only report waiting messages.

Skip this if your Jobs start instantly or you're already keeping extra capacity with `minReplicaCount`. The `eager` strategy also makes this less critical since it maximizes capacity anyway.

---

## Multiple Scalers Calculation

```yaml
scalingStrategy:
  multipleScalersCalculation: "max"  # Optional. Default: max
```

When you define multiple triggers for a ScaledJob, KEDA needs to decide which metric to use for scaling decisions. This setting determines how those metrics are combined.

**max** (default) - Use the scaler reporting the highest queue length. This ensures you scale up enough to handle the busiest source.

**min** - Use the scaler reporting the lowest queue length. This is more conservative and prevents over-scaling when sources are imbalanced.

**avg** - Average the queue lengths from all active scalers. This smooths out variations between different sources.

**sum** - Add together all scaler queue lengths. Use this when each scaler represents independent work that all needs processing.

Choose based on your workload: if triggers represent different views of the same work, use `max`. If they represent independent work streams, use `sum`.
