# CFP-30788: Managing Hubble Metrics cardinality

**SIG: SIG-HUBBLE-API/Observability**

**Begin Design Discussion:** 2024-03-28

**Cilium Release:** -

**Authors:** Vamsi Kalapala <vakr@microsoft.com>, Neha Aggarwal <neaggarw@microsoft.com>

## Summary

Hubble metrics are very powerful and can be used to monitor a wide variety of network traffic. However, the cardinality of the metrics can grow very quickly and can lead to performance issues. This CFP proposes a mechanism to manage the cardinality of Hubble metrics.

## Motivation

Consider a large scale cluster with thousands of pods in a Cloud Environment. Cloud environment managed prometheus offerings have costs associated at multiple levels, including storage, query, and ingestion. With auto-scaling enabled in the cluster and need for in-depth monitoring, the cardinality of the metrics can grow very quickly and can lead to performance, cost, and scalability issues.

Usually, Cluster admins would be able to control cardinality of the metrics ingested by applying custom ingestion rules on the prometheus configuration. However, there are scenarios where this may not be ideal, for example:

- When the cluster is using a managed prometheus offering where the prometheus configuration is managed by the cloud provider and custom ingestion rules cannot be applied.
- When a high scale cluster is generating high cardinality of metrics putting pressure on the prometheus ingestion, storage and query performance.
- When the cluster admin wants to dynamically control the cardinality of the metrics based on the cluster state without having to restart cilium agent or prometheus server. 

Such issues present a unique challenge to cluster admins and developers. By creating a mechanism to manage the cardinality of Hubble metrics dynamically based on the cluster state would help in addressing the above challenges. Recent change for [Hubble Flows dynamic exporter](https://docs.cilium.io/en/latest/observability/hubble-exporter/) has inspired this proposal.

## Goals

* Ability to manage the cardinality of Hubble metrics dynamically.
* Should be able to control labels and dimensions of the metrics.


## Non-Goals

* Recommendations for Prometheus configuration at ingestion time.

## Proposal

### Overview

We propose a new config map wrapped functionality in Hubble metrics which can be used to manage the cardinality, dimensions and targets of the metrics. Drawing from [Hubble Flows dynamic exporter](https://docs.cilium.io/en/latest/observability/hubble-exporter/) which uses a standalone config map to manage exporting of flows, we propose a similar mechanism. 

Existing behavior of reading cilium-config for `hubble-metrics` be preserved for backward compatibility. 

A new config on cilium agent called `enable-dynamic-hubble-metrics` will be introduced which when set to true, will enable the dynamic management of Hubble metrics. When this config is set to true, the cilium agent will read the `hubble-metrics-config` config map to get the configuration for the metrics. When this config is set to false, the cilium agent will read the `cilium-config` for `hubble-metrics` configuration.

### Config Map

A new config map called `hubble-metrics-config` will be introduced which will have the following fields:

- `metrics`: List of metrics to be managed. If no metrics are defined in this config map then essentially Hubble should not export any metrics. Each metric will have the following fields:
  - `name`: Name of the metric.
  - `contextOptions`: List of [context options](https://docs.cilium.io/en/latest/observability/metrics/#context-options) to be applied to the metric.
  - `includeFilters` : List of include flow filters to be applied to this particular metric.
  - `excludeFilters` : List of exclude flow filters to be applied to this particular metric.

### Example
    
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
    name: hubble-metrics-config
    namespace: kube-system
data:
    hubblemetrics.yaml: |
        hubbleMetrics:
            metrics:
            - name: flow
                contextOptions: 
                    - name: sourceEgressContext
                      values:
                        - pod
                    - name: destinationIngressContext
                      values:
                        - pod
                includeFilters: 
                    - []
                excludeFilters:
                    - source_pod: ["namespace1/"]
                    - destination_pod: ["kube-system/"]
            - name: dns
                contextOptions: 
                    - name: labelsContext
                      values:
                        - source_namespace
                        - source_pod
                    - name: destinationIngressContext
                      values:
                        - pod
                includeFilters: 
                    - source_pod: ["namespace1/"]
```

This will make sure we will have a dynamic way to manage each metric's cardinality, dimensions and targets.


## Impacts / Key Questions

### Impact: Managing Hubble Metrics Cardinality

This proposal intents to give more control of metrics to the cluster admins and developers. This will help in managing the cardinality of the metrics dynamically based on the cluster state without having to restart cilium agent or prometheus server. 

### Impact: Dashboarding and Alerting

### Will this proposal affect the existing dashboards and alerting rules?

By keeping the existing behavior of reading cilium-config for `hubble-metrics` for backward compatibility, this proposal will not affect the existing dashboards and alerting rules.

### Will this proposal affect dashboards built using the dynamic metrics config?

Yes there is a possibility that the dashboards built using the dynamic metrics config may be affected if the properties of the metrics are changed dynamically. This will be a trade-off between the dynamic management of metrics and the stability of the dashboards. The cluster admins and developers should be aware of this trade-off.

Also, this problem exists even with the existing behavior of reading cilium-config for `hubble-metrics`. New context options could be added to the metrics which may not be supported by the existing dashboards. 

## Implementation Alternatives

### Option 1: Using CRD approach instead of ConfigMaps 

CRDs are good alternative to approach solving this problem, there was good amount of discussion done previously [here](https://github.com/cilium/cilium/pull/28220#issuecomment-1741109225). For this proposal I built upon the existing precedence so there is uniformity on how Hubble is configured for dynamic flow logs and metrics.

#### Pros

* Versioned and changes are preserved. 
* Can implement complex validation logic

#### Cons

* Unlike ConfigMaps, CRDs are heavy-weight. In real-world scenarios, my assumption is that the dynamic hubble metrics config map will not be changed often, so CRD based continuous watcher might be overkill.
* Based on existing precedence set by Hubble dynamic flow logs configmap, users of Cilium do not have a re-learn a new way of configuring Hubble via CRD. 

Open to discuss more on this. 

### Option 1: Move only the flow filters to the dynamic metrics config


#### Pros

* This will reduce the re-learning aspect of existing way to configure hubble metrics.

#### Cons

* It creates a split brain scenario where some of the same metric properties are configured in cilium-config and some in hubble-metrics-config.

### Option 3: Introduce annotations to manage the cardinality of the metrics

#### Pros

* Easy to apply the annotations to the pods/namespaces

#### Cons

* No central place to manage which pods/namespaces have the annotations applied.


## Future Milestones

### Deprecation of existing hubble-metrics in cilium-config

This proposal is focusing on discussing new way of configuring hubble-metrics out-of-band from cilium-config. Deprecation of existing configuration method is necessary to reduce different code paths and complexity in configuring Hubble metrics. Current consensus is to re-visit deprecation of hubble metrics in cilium-config after 2 major versions of cilium. This gives enough time for new configMap approach to be validated and tested enough.