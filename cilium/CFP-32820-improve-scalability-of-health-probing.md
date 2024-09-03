# CFP-32820: Improve Scalability of Health Probing

**SIG: SIG-Scalability**

**Begin Design Discussion:** 2024-08-12

**Cilium Release:** 1.17

**Authors:** Shreya Jayaraman <shjayaraman@microsoft.com> based on the work of Marcel Zięba <marcel.zieba@isovalent.com> and feedback from Joe Stringer <joe@cilium.io>

**Status:** Implementable

## Summary

Cilium currently recommends disabling ICMP and HTTP health probes for large-scale clusters due to their impacts on cpu/mem usage. We propose to dynamically set the probing interval and thereby provide a configurable number of queries per second issued by probers to reduce the effects of ping frequency at scale.

## Motivation

Currently, Cilium uses runs ICMP and HTTP probes at intervals of 60 seconds, which may be an issue in large-scale clusters and is why Cilium currently [recommends that users disable health checking](https://docs.cilium.io/en/stable/operations/performance/scalability/report/) in such clusters.

There is a trade-off between resource consumption (for e.g., cpu/mem usage on the cluster and network bandwidth) and the frequency of updates regarding connectivity status. The method of health probing proposed here intends to allow Cilium users to determine which is more important to them.

### User Journeys

There are two conflicting user profiles to testing cluster connectivity which this CFP aims to address:

1. As a cluster administrator, I have a large cluster and want to ensure that the network bandwidth consumed for validating basic connectivity across nodes is well-bounded, even if it means that Cilium can less quickly tell when there is a connectivity breakage on the underlying network.

2. As a cluster administrator, I want to be able to quickly tell whether basic connectivity is OK across nodes, potentially even at the expense of network bandwidth usage within the environment.

This proposal intends to provide users with a way to find a balance between probe frequency and resource use that reflects their needs.

## Background

In the current health probing system, there are ICMP and HTTP probes configured to probe nodes and health endpoints.

### ICMP Probe

The ICMP probe currently in-use is the [servak/go-fastping](https://github.com/servak/go-fastping) library. This probe sends a burst of pings to all the node IPs and health endpoint IPs with intervals of MaxRTT seconds, where MaxRTT is currently set to [60 seconds](https://github.com/cilium/cilium/blob/6186d579ed60f334c7a4daaf81060797b02cc6bd/cilium-health/launch/launcher.go#L37).

### HTTP Probe

The HTTP probe also runs at intervals of 60 seconds. In every loop, the probe will send an HTTP GET request to the /hello endpoint of every IP [known](https://github.com/cilium/cilium/blob/6186d579ed60f334c7a4daaf81060797b02cc6bd/pkg/health/server/prober.go#L260) to the probe.

As every node in the cluster or clustermesh probes every other node and their health endpoints, probing is in the order of O(n^2). Further, the ICMP probe bursts pings to every node and the HTTP probe loops and probes a health endpoint on each node.

## Proposal

As an alternative, we propose the following updated design for health checking.

### Configurable and Auto-Scaling Probing Interval

First, the default probing interval will be adjusted so that it is no longer a constant 60 seconds and instead, a value that is dependent on cluster size. For example, for a cluster of size `n`, i.e., with `n` discovered nodes, the probing interval will be set to `N = ClusterSizeDependentInterval(n, k)`, which returns a value dependent on the cluster size according to the [function](https://github.com/cilium/cilium/blob/6186d579ed60f334c7a4daaf81060797b02cc6bd/pkg/backoff/backoff.go#L127) `k * ln(1+n)`.

The value of `k` represents the base interval value and will be configurable, to a value between 10 and 110.

For reference, the following table shows a back-of-the-napkin estimate for the probes per second and bandwidth consumed by ICMP pings, assuming a standard ICMP packet size of 74 bytes.

| Cluster Size `n` | Base Interval `k` | Probe Interval `i=k*ln(1+n)` (s)| ICMP Events per Second `n^2 / i` | Estimate for Bandwidth for ICMP Pings (Kbps) |
| ------------ | ------------- | ------------------ | ----------------- | -------------------------------------------- |
| 100 | 10 | 46 | 217 | 128 |
| 1000 | 10 | 69 | 14474 | 8568 |
| 5000 | 10 | 85 | 293517 | 173762 |
| 100 | 60 | 276 | 36 | 21 |
| 1000 | 60 | 414 | 2412 | 1428 |
| 5000 | 60 | 511 | 48920 | 28960 |
| 100 | 110 | 508 | 20 | 12 |
| 1000 | 110 | 759 | 1316 | 779 |
| 5000 | 110 | 937 | 26683 | 15796 |

The minimum value of `k=10` is similar to the current interval, and may be favored by users who require health probe metrics to be updated more frequently or have smaller clusters. On the other hand, setting `k=110` will cause very infrequent probing, which could benefit users who don't require such frequent metrics updates or who have larger clusters.

The value of `k` will be indirectly configurable by the user, using the formula `k = 10 +  health-probe-frequency-ratio * 100`, where `--health-probe-frequency-ratio` is a cilium-agent flag that is a float in the interval `[0,1]`. 0 will give the most frequent probing and 1 will give the least frequent probing. Note that due to the function used, it is safe for the user to pass in a value of 0 or 1. We will set the default value of health-probe-frequency-ratio to be 0.5 to balance probing frequency with CPU and memory usage. (Note: during implementation, some testing can be done to ensure these initially proposed values are sensible for most supported cluster sizes.)

In the current design, the health server discovers nodes in the cluster after the probe interval passes. This design should be updated so that every time the server gets the nodes in the cluster, the probe interval is also updated using the updated cluster size. Note that the updated probe interval should not affect the current sleep duration, but only the next one, i.e., the health server will:

```
server.runActiveServices:
	Get the current node set, run synchronous probe, update metrics.
	Get node set, initialize async probe.
	In a loop:
        Start sending out ICMP/HTTP probes to known node set.
		Wait for probe interval based on known node set.
		OnIdle(), update known node set and clean up stale metrics, and set probe interval based on new known node set.
```

### Queries Per Second

We set the probe interval to `N` for a cluster with n nodes. Then, the implied number of queries per second that ICMP and HTTP probes are each limited to is `floor(N/n)`, except in the last interval, where if `N%n` is non-zero, there will be `N%n` queries left. The querying may be uneven in the last interval, but as the user has not been promised a particular number of queries per second, we are not breaking a guarantee.

### Probe Configuration & Design

#### ICMP Probe
The ICMP probe is currently configured to run via a [call](https://github.com/cilium/cilium/blob/4e012f493ed7eda2a745be9dd22db5503ed06765/pkg/health/server/prober.go#L393) to a non-blocking method which sends packets to all IPs added in a burst, at intervals of MaxRTT. In this design, we may want to replace the currently in-use package servak/go-fastping with, for e.g., [prometheus-community/pro-bing](https://github.com/prometheus-community/pro-bing). This library does not have a RunLoop() function that bursts pings to all added IPs. Instead, we could write our own loop that creates ICMP Pingers per IP and use [time/rate](https://pkg.go.dev/golang.org/x/time/rate) to send ICMP pings in accordance with the configured requests per second. More details on why we may find it easier to change the library used for the ICMP probe are in the section `Other Details / Probe Timeouts and Errors`.

#### HTTP Probe

The HTTP probe runs in a for loop and [probes each IP](https://github.com/cilium/cilium/blob/fac90aa858c86eefd2c4bdccef8b56abf24a7b58/pkg/health/server/prober.go#L349). Again, for consistency in design, we may modify this probe using [time/rate](https://pkg.go.dev/golang.org/x/time/rate) to run the probe in accordance with the configured requests per second.

### Other Details

#### Connectivity Status Cache Update
The connectivity cache is currently updated on the condition that the current health report start time is before the health report being cached, at the end of the ProbeInterval period.

With the ProbeInterval set to longer than 60s, it may make more sense to update the connectivity cache more frequently than the ProbeInterval, so that users see some refreshed metrics based on what has been collected at a given point.

A possible solution for this may be to continue refreshing the connectivity cache at intervals of 60 seconds. Instead of storing the start time for the health report as a whole, we may store the health report start time on a per-node basis. Then, every 60 seconds, we make a call to refresh the cache, and update metrics for nodes iff the start time for the node’s last stored metric is before the start time of the current health report.

#### Metrics Update Period

As a consequence of this proposal, health probe updates may take longer and be more unpredictably-timed for users. In order to keep users aware of how long it will take to receive accurate metrics for the known nodes in the cluster, we can update the output of `cilium-health status` to also report the current probe interval and/or the most recent timestamp at which metrics for all known nodes were updated.

#### Probe Timeouts and Errors

The HTTP probe is currently set to have a timeout of 30 seconds. The ICMP probe has MaxRTT set to 60 seconds, which acts as the timeout for the pings. Both timeouts seem larger than necessary. The HTTP probe timeout can be re-configured to 10 seconds, which is a reasonable timeout and accounts for the sizes of the maximum bucket configured for the histogram metric `node_health_connectivity_latency_seconds`.

However, for the ICMP probe, by nature of the library `servak/go-fastping`, the probe interval value and max RTT cannot easily be separated. This is one reason to consider changing to the library `prometheus-community/pro-bing`, which will allow us to easily configure different Timeout and Interval values, allowing us to set the timeout to 10 seconds and the interval to the probe interval value.

#### Synchronous Probe Behavior
Currently, a synchronous probe is run in two situations:
1. When the health server starts running
2. When an HTTP GET request is made to the health server for /status/probe, usually by a call to `cilium-health status --probe`.

We will plan to deprecate the capability to synchronously probe the health server via the `cilium-health` CLI command. This will reduce the negative effects of synchronous probing on resource usage at high-scale. On start-up, probing will rely on the probe interval determined by cluster size, as in the typical case.

## Key Questions

### Key Question:

What is the preferable approach to configuring the QPS?

### Option 1 (Selected):

Configure the flag `--connectivity-probe-frequency-ratio` described in this proposal.

#### Pros

* This flag abstracts away decision-making regarding the base interval from the user, and instead allows the user to make a decision regarding the trade-off between probe frequency and resource usage with a well-defined range of possible values and ease-of-understanding of a ratio between 0 and 1.

#### Cons

* The obfuscation may be more complex to interpret and document.
* Removes some direct configuration power from user.

### Option 2 (Discarded):

Allow user to configure base interval directly for ClusterSizeDependentInterval.

#### Pros

* Simpler to communicate to user.

#### Cons

* May be less intuitive for users to understand the impact of a particular base interval on the probing interval and cluster size.

### Option 3 (Discarded):

Allow user to directly configure QPS, using flags such as `--health-probe-icmp-qps` and `--health-probe-http-qps`.

#### Pros

* Could be easier to understand exactly what is being configured.

#### Cons

* Will not dynamically scale with cluster size.

