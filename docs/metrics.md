# Mayastor Metrics

This document describes the metrics exposed by Mayastor components for monitoring with Prometheus.

## Disk Pool Metrics

Disk pool metrics are exposed by the io-engine metrics exporter running as a sidecar container.

| Metric name                | Metric type | Labels/tags | Metric unit | Description                                                                    |
|----------------------------| ----------- | ----------- | ----------- |--------------------------------------------------------------------------------|
| disk_pool_total_size_bytes | Gauge | `name`=&lt; pool_id&gt; <br> `node`=&lt;pool_node&gt; | Integer | Total size of the pool                                                         |
| disk_pool_used_size_bytes  | Gauge | `name`=&lt; pool_id&gt; <br> `node`=&lt;pool_node&gt; | Integer | Used size of the pool                                                          |
| disk_pool_status           | Gauge | `name`=&lt; pool_id&gt; <br> `node`=&lt;pool_node&gt; | Integer | Status of the pool (0, 1, 2, 3) = {"Unknown", "Online", "Degraded", "Faulted"} |

## Node Status Metrics

Node status metrics are exposed by the control-plane REST API at the `/v0/metrics` endpoint.

| Metric name                | Metric type | Labels/tags | Metric unit | Description                                                                    |
|----------------------------| ----------- | ----------- | ----------- |--------------------------------------------------------------------------------|
| mayastor_node_status       | Gauge | `node`=&lt;node_id&gt; | Integer | Status of mayastor node (0=Unknown, 1=Online, 2=Offline)                      |
| mayastor_node_cordoned     | Gauge | `node`=&lt;node_id&gt; | Integer | Whether mayastor node is cordoned (0=No, 1=Yes)                               |
| mayastor_node_drain_state  | Gauge | `node`=&lt;node_id&gt; | Integer | Drain state of node (0=None, 1=Cordoned, 2=Draining, 3=Drained)              |

### Example Disk Pool Metrics:

```
# HELP disk_pool_status mayastor name status
# TYPE disk_pool_status gauge
disk_pool_status{node="worker-0",name="mayastor-disk-pool"} 1
# HELP disk_pool_total_size_bytes mayastor name total size in bytes
# TYPE disk_pool_total_size_bytes gauge
disk_pool_total_size_bytes{node="worker-0",name="mayastor-disk-pool"} 5.360320512e+09
# HELP disk_pool_used_size_bytes mayastor name used size in bytes
# TYPE disk_pool_used_size_bytes gauge
disk_pool_used_size_bytes{node="worker-0",name="mayastor-disk-pool"} 2.147483648e+09
```

### Example Node Status Metrics:

```
# HELP mayastor_node_status Status of mayastor node (0=Unknown, 1=Online, 2=Offline)
# TYPE mayastor_node_status gauge
mayastor_node_status{node="worker-0"} 1
mayastor_node_status{node="worker-1"} 1
mayastor_node_status{node="worker-2"} 2
# HELP mayastor_node_cordoned Whether mayastor node is cordoned (0=No, 1=Yes)
# TYPE mayastor_node_cordoned gauge
mayastor_node_cordoned{node="worker-0"} 0
mayastor_node_cordoned{node="worker-1"} 1
mayastor_node_cordoned{node="worker-2"} 0
# HELP mayastor_node_drain_state Drain state of mayastor node (0=None, 1=Cordoned, 2=Draining, 3=Drained)
# TYPE mayastor_node_drain_state gauge
mayastor_node_drain_state{node="worker-0"} 0
mayastor_node_drain_state{node="worker-1"} 2
mayastor_node_drain_state{node="worker-2"} 0
```

## Accessing Metrics

### IO-Engine Metrics (Disk Pools)
IO-engine metrics are exposed on each node running the io-engine daemonset:
- Default endpoint: `http://<node-ip>:9502/metrics`
- Cached data refreshed every 5 minutes

### Control-Plane Metrics (Node Status)
Node status metrics are exposed by the REST API service:
- Endpoint: `https://<rest-api-service>:8080/v0/metrics` or `http://<rest-api-service>/v0/metrics` (if HTTP is configured)
- Metrics are refreshed on each request

## Prometheus Configuration

Example Prometheus scrape configuration for both metric sources:

```yaml
scrape_configs:
  # IO-Engine metrics (disk pools)
  - job_name: 'mayastor-io-engine'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: io-engine
      - source_labels: [__meta_kubernetes_pod_container_port_number]
        action: keep
        regex: "9502"
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
    scrape_interval: 30s

  # Control-Plane metrics (node status)
  - job_name: 'mayastor-control-plane'
    kubernetes_sd_configs:
      - role: service
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: mayastor-api-rest
      - source_labels: [__meta_kubernetes_service_port_name]
        action: keep
        regex: http|https
    scrape_interval: 30s
    scheme: https
    tls_config:
      insecure_skip_verify: true
```

## Alerting Rules

Example Prometheus alerting rules using the node status metrics:

```yaml
groups:
  - name: mayastor_node_alerts
    rules:
      - alert: MayastorNodeOffline
        expr: mayastor_node_status == 2
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Mayastor node {{ $labels.node }} is offline"
          description: "Mayastor node {{ $labels.node }} has been offline for more than 5 minutes."

      - alert: MayastorNodeCordoned
        expr: mayastor_node_cordoned == 1
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Mayastor node {{ $labels.node }} is cordoned"
          description: "Mayastor node {{ $labels.node }} has been cordoned for more than 15 minutes."

      - alert: MayastorNodeDraining
        expr: mayastor_node_drain_state == 2
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Mayastor node {{ $labels.node }} is draining"
          description: "Mayastor node {{ $labels.node }} has been in draining state for more than 30 minutes."
```