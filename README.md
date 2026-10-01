# Vector: Shared Kubernetes Metadata Watches

This fork adds a process-local cache for Kubernetes metadata watches used by the `kubernetes_logs` source. Sources with compatible watch parameters can share Pod, Namespace, and Node metadata streams inside one Vector process.

The change reduces duplicate Kubernetes API watches while keeping each source's log reader, checkpoint, transform path, and sink independent. Cache compatibility includes the Kubernetes client identity, field selector, label selector, and `use_apiserver_cache` setting; sources with different selectors do not share an incompatible watch.

## Why this fork exists

With many `kubernetes_logs` sources, each source can create its own Pod, Namespace, and Node LIST/WATCH streams. This multiplies API watches across sources and Vector agents. The fork reuses metadata streams for sources that request the same Kubernetes view, without combining their downstream delivery paths.

## Measured Docker Desktop lab

The measurements below come from a controlled `docker-desktop` Kubernetes lab run on 2026-09-30. The deployment had four Vector agent Pods and generated Kubernetes workloads. The baseline image was built from the upstream base revision; the fork image included this branch's shared metadata cache.

Each measured phase configured 20, 100, or 170 `kubernetes_logs` pipelines per agent. The fork's primary comparison used 170 pipelines with one common selector. A separate fork run used 30 selector values to check that selector differences remained isolated.

| Run | Pipelines per agent | Pod / Namespace / Node WATCH | Vector CPU | Vector memory | Healthy receiver rate | API p99 | API errors |
|---|---:|---:|---:|---:|---:|---:|---:|
| Baseline | 20 | 100 / 93 / 101 | 0.35 cores | 590 MB | 151 events/s | 19.9 ms | 0 |
| Baseline | 100 | 420 / 413 / 421 | 1.23 cores | 10.8 GB | 795 events/s | 19.1 ms | 0 |
| Baseline | 170 | 700 / 694 / 703 | 1.92 cores | 19.0 GB | 1,356 events/s | 20.7 ms | 0 |
| Fork, one selector | 170 | 24 / 18 / 25 | 1.90 cores | 3.10 GB | 1,371 events/s | 23.3 ms | 0 |
| Fork, 30 selectors | 170 | 136 / 17 / 25 | 1.89 cores | 3.37 GB | 1,387 events/s | 22.0 ms | 0 |
| Fork, recovered | 170 | 22 / 15 / 25 | 1.72 cores | 3.65 GB | 1,385 events/s | 22.6 ms | 0 |

For the direct 170-pipeline comparison, Pod, Namespace, and Node WATCH totals fell from **2,097 to 67**, a **96.8% reduction**. The total Kubernetes WATCH count also includes unrelated controllers, so it is not expected to equal the sum of these three resource series. The receiver continued to accept a similar event rate in both runs. API p99 was slightly higher in the fork snapshots; this run does not show an API latency improvement.

With 30 selector values, Pod WATCH increased from 24 to 136 while Namespace and Node WATCH remained near 17 and 25. This is consistent with Pod watches remaining distinct for differing selectors while compatible Namespace and Node metadata watches are reused.

## Deployment and measurements

The lab deployed baseline and fork Vector images as four Kubernetes agents. A load generator created the configured `kubernetes_logs` sources and independent HTTP sinks; a local receiver counted accepted events and could return HTTP 503 for one selected sink. The fork image ran 170 sources and 170 HTTP sinks per agent in the principal comparison.

The lab collected:

- Kubernetes API server active WATCH counts by resource, API request errors, and request latency.
- Vector process CPU and memory, component error counters, and HTTP sink throughput and buffer occupancy.
- Receiver accepted event counts, grouped into healthy and failing destinations.
- Run markers and timestamped JSON snapshots for each phase.

VictoriaMetrics retained the metrics, and Grafana displayed the phase comparisons. The snapshots and phase markers are kept with the lab artifacts; they are not included in this source repository.

## Sink failure and recovery

During a 125-second test, the receiver returned 503 only for sink `p0000`. That sink's accepted rate fell to zero and its four per-agent buffers reached 400/400 events in total. Healthy receiver throughput continued at about 1,356 events/s; Kubernetes API errors remained at zero and WATCH stayed near 311. After the failure was disabled, the queued events drained and `p0000` returned to its normal rate.

This verifies that backpressure in the failing sink did not stop accepted delivery to the healthy sinks in this lab run. The test did not audit event IDs, so it is not a guarantee of lossless delivery.

## Scope and limitations

These are measurements from one Docker Desktop lab run, not a production deployment or a soak test. The run does not establish behavior at 10,000 real Pods. CPU and memory values are snapshots of the lab workload; API p99 did not improve in the fork snapshots. Selector diversity affects how many metadata streams can be shared. Further validation should include longer runs, selector churn, cache reconnect and termination behavior, and broader resource and delivery checks.

## Implementation

The cache implementation is in [`src/sources/kubernetes_logs/shared_metadata.rs`](src/sources/kubernetes_logs/shared_metadata.rs), with source integration in [`src/sources/kubernetes_logs/mod.rs`](src/sources/kubernetes_logs/mod.rs). It is process-local and shares metadata watching only; source readers, checkpoints, transforms, and sinks remain separate.
