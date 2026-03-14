# Metrics

## VPA-based resource recommendations

- Based on [Vertical Pod Autoscaler (VPA): The Recommender](https://erikzilinsky.com/posts/vpa1.html).
- For layout, see [How to Manage Kubernetes Resources and Costs with Grafana Cloud | Grafana](https://www.youtube.com/watch?v=OymWTzpl_Bk).

### Variables

- Namespace: (Query)

  ```promql
  label_values(kube_namespace_status_phase{job="integrations/kubernetes/kube-state-metrics"},namespace)
  ```

- Pod: Text, e.g. `my-deployment.*`
- Container: (Query)

  ```promql
  label_values(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod"},container)
  ```

For multi-cluster support, use the `up` metric. For example:

```promql
label_values(up{job="integrations/kubernetes/kube-state-metrics"},cluster)
```

### Usage

#### Container CPU

- Visualisation: Time series
- Description: Container CPU usage over time.
- Unit: Cores (Custom)

```promql
# Max Used
max(rate(container_cpu_usage_seconds_total{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
}[$__rate_interval]))

# Request
max(kube_pod_container_resource_requests{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="cpu"
})

# Limit
max(kube_pod_container_resource_limits{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="cpu"
})
```

#### Container Memory

- Visualisation: Time series
- Description: Container memory usage over time.
- Unit: Cores (Custom)

```promql
# Max Used
max(container_memory_working_set_bytes{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
})

# Request
max(kube_pod_container_resource_requests{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="memory"
})

# Limit
max(kube_pod_container_resource_limits{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="memory"
})
```

### Recommendations

#### Current (CPU Limit)

- Visualisation: Stat
- Description: The current CPU limit.
- Calculation: Last *
- Unit: Cores (Custom)

```promql
max(kube_pod_container_resource_limits{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="cpu"
})
```

#### CPU Limits Sizing

- Visualisation: Gauge
- Description: The difference between the current and recommended CPU limit. The acceptable tolerance is ±0.2 cores.
- Calculation: Last *
- Unit: Cores (Custom)
- Min: -2
- Max: 2
- Thresholds:
    - Base: Red
    - -0.2: Green
    - 0.2: Red
 
```promql
max(kube_pod_container_resource_limits{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="cpu"
}) - quantile_over_time(0.99, max(rate(container_cpu_usage_seconds_total{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
}[30m]))[7d:]) * 1.15
```

#### Recommended (CPU Limit)

- Visualisation: Stat
- Description: The recommended CPU limit, calculated as P99 of the max used across 7 days, plus 15%.
- Calculation: Last *
- Unit: Cores (Custom)

```promql
quantile_over_time(0.99, max(rate(container_cpu_usage_seconds_total{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
}[30m]))[7d:]) * 1.15
```

#### Current (CPU Request)

- Visualisation: Stat
- Description: The current CPU request.
- Calculation: Last *
- Unit: Cores (Custom)

```promql
max(kube_pod_container_resource_requests{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="cpu"
})
```

#### CPU Requests Sizing

- Visualisation: Gauge
- Description: The difference between the current and recommended CPU request. The acceptable tolerance is ±0.2 cores.
- Calculation: Last *
- Unit: Cores (Custom)
- Min: -2
- Max: 2
- Thresholds:
    - Base: Red
    - -0.2: Green
    - 0.2: Red

```promql
max(kube_pod_container_resource_requests{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="cpu"
}) - quantile_over_time(0.5, max(rate(container_cpu_usage_seconds_total{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
}[30m]))[7d:]) * 1.15
```

#### Recommended (CPU Request)

- Visualisation: Stat
- Description: The recommended CPU request, calculated as P50 of the max used across 7 days, plus 15%.
- Calculation: Last *
- Unit: Cores

```promql
quantile_over_time(0.5, max(rate(container_cpu_usage_seconds_total{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
}[30m]))[7d:]) * 1.15 
```

#### Current (Memory Limit)

- Visualisation: Stat
- Description: The current memory limit.
- Calculation: Last *
- Unit: Bytes (IEC)

```promql
max(kube_pod_container_resource_limits{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="memory"
})
```

#### Memory Limits Sizing

- Visualisation: Gauge
- Description: The difference between the current and recommended memory limit. The acceptable tolerance is ±100MiB.
- Calculation: Last *
- Unit: Bytes (IEC)
- Min: -1073741824 (1GiB)
- Max: 1073741824 (1GiB)
- Thresholds:
    - Base: Red
    - -104857600 (100MiB): Green
    - 104857600 (100MiB): Red

```promql
max(kube_pod_container_resource_limits{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="memory"
}) - quantile_over_time(0.99, max(container_memory_working_set_bytes{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
})[7d:]) * 1.15
```

#### Recommended (Memory Limit)

- Visualisation: Stat
- Description: The recommended memory limit, calculated as P99 of the max used across 7 days, plus 15%.
- Calculation: Last *
- Unit: Bytes (IEC)

```promql
quantile_over_time(0.99, max(container_memory_working_set_bytes{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
})[7d:]) * 1.15
```

#### Current (Memory Request)

- Visualisation: Stat
- Description: The current memory request.
- Calculation: Last *
- Unit: Bytes (IEC)

```promql
max(kube_pod_container_resource_requests{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="memory"
})
```

#### Memory Requests Sizing

- Visualisation: Gauge
- Description: The difference between the current and recommended memory request. The acceptable tolerance is ±100MiB.
- Calculation: Last *
- Unit: Bytes (IEC)
- Min: -1073741824 (1GiB)
- Max: 1073741824 (1GiB)
- Thresholds:
    - Base: Red
    - -104857600 (100MiB): Green
    - 104857600 (100MiB): Red

```promql
max(kube_pod_container_resource_requests{
  namespace=~"$namespace",
  pod=~"$pod",
  container=~"$container",
  resource="memory"
}) - quantile_over_time(0.5, max(container_memory_working_set_bytes{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
})[7d:]) * 1.15
```

#### Recommended (Memory Request)

- Visualisation: Stat
- Description: The recommended memory request, calculated as P50 of the max used across 7 days, plus 15%.
- Calculation: Last *
- Unit: Bytes (IEC)

```promql
quantile_over_time(0.5, max(container_memory_working_set_bytes{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
})[7d:]) * 1.15
```

### Issues

#### CPU Throttling

- Visualisation: Time series
- Unit: Percent (0.0-1.0)

```promql
sum(increase(container_cpu_cfs_throttled_periods_total{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
}[$__rate_interval])) / sum(increase(container_cpu_cfs_periods_total{
  namespace=~"$namespace",
  pod=~"$pod",
  container!="",
  container=~"$container"
}[$__rate_interval]))
```
