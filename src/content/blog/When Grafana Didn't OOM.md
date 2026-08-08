---
title: When Grafana Didn't OOM
description: Grafana became severely unresponsive and its container restarted.
pubDate: 2026-08-08
updatedDate: 2026-08-08
tags:
  - Kubernetes
  - Reliability
draft: false
---
## Summary

On August 8, 2026, the Grafana instance in my K8s cluster became increasingly slow and eventually restarted. My initial suspicion was an out of memory (OOM) event because at the time Grafana had a `512Mi` memory limit defined in my `helmrelease.yaml` and its memory usage was high.

As it turned out, that hypothesis was in the right direction but not true or complete.

The container was **not `OOMKilled`**. Grafana received `SIGTERM` and exited cleanly with exit code `0`. Containerd logs showed an explicit `StopContainer` request immediately before restart. Kubelet probe metrics showed roughly ten liveness probe failures in the incident window which is the same value configured for `failureThreshold`.

At the same time, `container_memory_usage_bytes` showed Grafana rise sharply to approximately its `512Mi` limit and remain there until its eventual restart. Therefore, the evidence supports a more subtle failure mode. Grafana came under severe memory pressure, became too unresponsive to satisfy its liveness probe, and K8s restarted it.

The restart mechanism is well-supported by the evidence. Memory pressure is the leading hypothesis for why Grafana became unhealthy, however I am treating that as a hypothesis to validate rather than simply claiming an OOM that never occurred.

---
## Environment

At the time of the incident, the relevant configuration was:
```yaml
resources:
	requests:
		cpu: 100m
		memory: 256Mi
	limits:
		memory: 512Mi
```

The liveness probe used `/api/health` with:
```
periodSeconds: 10
timeoutSeconds: 30
failureThreshold: 10
```

The affected Pod was:
	`grafana-8dc5f8587-6zcjm`
On:
	`k8s-worker-01`

---
## What I Observed

Grafana initially worked normally after accessing the Web UI. After remaining on a dashboard for approximately two minutes, it would become extremely sluggish before eventually timing out. The Grafana container restarted while the Pod itself remained alive.

That distinction matters: Kubernetes can restart an individual container inside an existing Pod without replacing the Pod itself.

The old Grafana container ID was:
	`c5fabfea68da1959d9ce90708eaeaf6553166c03e1e0d190760ad2b64b32b5f2`

The replacement was:
	`c230d731bec8777b0be160acc41778a986d3c48f9bc73c29ae00261efb347646`

---
## Investigation

### 1. I started with the obvious theory: OOM

Grafana had a `512Mi` memory limit, so an OOM kill was an obvious possibility.

I first examined cAdvisor's working-set metric:
```promql
max(
  max_over_time(
    container_memory_working_set_bytes{
      namespace="monitoring",
      container="grafana",
      name="c5fabfea68da1959d9ce90708eaeaf6553166c03e1e0d190760ad2b64b32b5f2"
    }[15m] @ 1786186945
  )
) / 1024 / 1024
```

The result was:
`461.8359365 MiB`

That was about 90% of the configured limit, but working-set memory is not the same thing as total cgroup memory usage. More importantly, the container status did not report `OOMKilled`. Grafana's own logs showed that it received a termination signal and exited normally.

That forced me to change the question from:

> Why did Grafana run out of memory?

To:

> What told containerd to terminate a still-running Grafana process?

### 2. Containerd exposed the termination path

The containerd journal contained the key sequence:
```
11:02:24 StopContainer for "c5fabfea..." with timeout 30 (s)
11:02:24 Stop container "c5fabfea..." with signal terminated
11:02:25 received container exit event
11:02:26 StopContainer ... returns successfully
11:02:26 CreateContainer ... name:"grafana" attempt:1
11:02:26 StartContainer for "c230d731..."
```

This ruled out an abrupt kernel OOM kill as the restart mechanism. Something in the `Kubernetes/container-runtime` path deliberately requested that the container stop. Containerd sent `SIGTERM`, Grafana shut down, and a new Grafana container was immediately created inside the same Pod sandbox.

Systemd also recorded the old container scope ending successfully:
```
cri-containerd-c5fabfea....scope: Deactivated successfully.
... 512.2M memory peak ...
```

The `512.2M` value reported by systemd should not be confused directly with Kubernetes' `512Mi` limit. While the displayed units are not equivalent, this was another signal that memory consumption deserved closer inspection.

### 3. Kubelet probe metrics supplied the missing evidence

The kubelet exposes a `prober_probe_total` counter on `/metrics/probes`. Kubernetes documents this as the cumulative number of liveness, readiness, or startup probes for a container broken down by result.

For the affected Grafana container, the counters showed:
```
Liveness failed:       29
Liveness successful:   32573
Readiness failed:      93
Readiness successful:  32566
```

The lifetime totals alone were not enough so I queried the increase around the incident:
```promql
increase(
  prober_probe_total{
    namespace="monitoring",
    pod="grafana-8dc5f8587-6zcjm",
    container="grafana",
    probe_type="Liveness",
    result="failed"
  }[10m]
)
```

Prometheus returned:
`10.5263157`

Obviously there cannot be a fractional probe failure and to explain the number, Prometheus extrapolates `increase()` to the boundaries of the requested range, so counter increases can produce non-integer results. In practical terms, this showed approximately ten failed liveness probes during the window.

That number was particularly important because the Grafana liveness probe was configured with:
```
failureThreshold: 10
```

Kubernetes restarts a container after its liveness probe reaches its configured `failureThreshold` of consecutive failures. The approximately ten failures observed during the incident aligned with my configured threshold of `10` and with the subsequent `StopContainer` event.

---
## Root Cause Analysis

The evidence strongly supports liveness probe failure as the immediate restart mechanism. Grafana accumulated approximately ten liveness probe failures during the incident window, aligning with its configured `failureThreshold: 10`. Immediately afterward, containerd received a `StopContainer` request, sent Grafana `SIGTERM`, and started a replacement container.

The underlying cause of Grafana becoming unhealthy appears to have been memory pressure. Total container memory rose sharply toward the configured `512Mi` limit at approximately 06:57 and remained elevated until the restart at 07:02.

I am treating memory pressure as the leading root cause hypothesis rather than simply claiming Grafana was OOM killed. It wasn't.

---
## Remediation

I increased Grafana's memory limit from:
```yaml
limits:
	memory: 512Mi
```
To:
```yaml
limits:
	memory: 1Gi
```

I intentionally left the liveness probe thresholds unchanged. The probe did exactly what it was designed to do: Grafana became unhealthy, and Kubernetes recovered the service.

---
## Validation

Following the change, I will monitor:
- Grafana memory utilization and working set
- Liveness and readiness probe failures
- Container restart count
- Dashboard/query latency
- Whether memory stabilizes or continues growing toward the new `1Gi` limit

If memory stabilizes above the previous `512Mi` limit and probe failures disappear, that will strengthen the memory pressure hypothesis.

If Grafana instead grows toward `1Gi`, increasing the limit will have exposed rather than solved the underlying problem, and further investigation will be needed.

---
## Takeaway

The biggest lesson from this incident was not to equate high memory usage and a container restart with `OOMKilled`.

Grafana exited with code `0` after receiving `SIGTERM`. Following the termination path through Grafana, containerd, kubelet probe metrics, and Prometheus revealed a different failure mode. The application became unhealthy under apparent memory pressure and K8s restarted it through its liveness mechanism.

That distinction changed the investigation from guessing at the most obvious cause to following the evidence.