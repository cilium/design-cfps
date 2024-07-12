# CFP-2024-06-12: Distributed Packet Tracing

## Meta
**SIG:**  
**Sharing:** Public  
**Begin Design Discussion:** Jun 12, 2024  
**End Design Discussion:**  
**Cilium Release:**  
**Authors:** Ben Bigdelle <bigdelle@google.com>, Mark St. John <markstjohn@google.com> based on the work of Nathan Perkins <nathanperkins@google.com>  
**Reviewers:**  

## Summary
We aim to enhance cross-cluster and multi-system observability for flows that contain an `ip_trace_id` IP Option (referred to as a tagged flow), thus supporting distributed tracing across networks. These tagged flows will trigger adding the ID value to Flow events messages and client (Hubble / Cilium Monitor) representation. This allows correlating flow events across systems which integrate with the span observability.

Additionally, within Cilium, these flows will generate datapath debugging events regardless of the global debug-verbose setting. This will improve debuggability of production systems.

## Motivation
There are currently limitations for gathering and correlating flow events:
- NAT transformations and DSR can all change the 4-tuple, so that a single Hubble or tcpdump filter cannot be used to trace the packet along the entire path – even when using a system such as Hubble Relay.
- Enabling verbose datapath debugging is not possible in production environments due to traffic volume which leads to dropping flows.

## Goals
- For tagged packets, add the ID to emitted flow events.
- For tagged packets, emit datapath debugging events regardless of the debug-verbose setting.
- Support both IPv4 and IPv6.
- Add hubble filter for trace ID.
- Allow users to define which flow IDs they want to monitor for events.

## Non-goals
- Automatically apply IP option to packets (see Future Milestones).

## User Stories
- As a platform administrator managing multiple clusters, users are reporting connection timeouts reaching destinations in remote clusters. Platform admin generates traffic with trace ID IP option along the problematic path to identify all flow events associated with that packet at both the source and destination.

![User story](./images/CFP-33716-user-story.png)

- As a production cluster administrator, particular flows from a client are experiencing connection timeouts (e.g. [issue #32472](https://github.com/cilium/cilium/issues/32472). Enabling datapath debugging for all flows leads to event loss and low SNR. To extract dataplane information to root cause the issue, generate traffic with trace ID IP option to emit datapath debug information specific to this flow.

## Proposal

### Overview
Offer support for packets containing a user-defined IP option field. For tagged incoming flows, handle events for this flow specially; emit datapath debugging events for these packets, include the ID in the event message, and implement hubble filtering by the ID.

### Section 1: Packet Tagging 
This feature does not implement the packet tagging (see Deferred Milestone 1: Tag Packets). Instead, it is the client's responsibility to tag packets via IP option embedding. Therefore, the key assumption for this feature is that tagged packets arrive at ingress points with an `ip_trace_id` already embedded in the IP options.

Automatic packet tagging is not implemented for several reasons:
- **Avoid MTU concerns:** Automatically tagging packets can cause them to exceed the MTU, leading to fragmentation and performance issues.
- **Prevent performance loss from GRO:** GRO engine does not support packets with IP options.
- **Avoid packet loss:** Some network devices and paths may not support IP options, causing tagged packets in the path to be dropped.
- **IP checksum recalc:** When IP options are added or modified, the IP checksum must be recalculated, adding additional processing overhead and complexity.

By making packet tagging the client's responsibility, we ensure that Cilium remains unaffected by these concerns, leaving it to the clients to incorporate these key considerations into their use of this feature and decide how best to utilize it to suit their specific needs.

### Section 2: Feature Enablement
`--enable-ip-option-tracing=<type>` to define the IP option type used to embed the trace ID value
- `<type>` corresponds to the IP option value ([https://www.iana.org/assignments/ip-parameters/ip-parameters.xhtml](https://www.iana.org/assignments/ip-parameters/ip-parameters.xhtml))
- By default, the feature is disabled (`<type>=0`).
- `<type>=0` disables this feature.

```go
EnableIPOptionTracing = "enable-ip-option-tracing"
```

```go
if option.Config.EnableIPOptionTracing > 0 {
    cDefinesMap["ENABLE_PACKET_IP_TRACING"] = 
    strconv.Itoa(option.Config.EnableIPOptionTracing)
}
```

## Section 3: Parsing

Parsing at the ingress point of the TC hook allows for early detection and tracing of a flow.

```c
#ifdef ENABLE_PACKET_IP_TRACING
    check_and_store_ip_trace_id(ctx, ENABLE_PACKET_IP_TRACING);
    // Do parsing and storing of ip_trace_id
#endif
```
```c
// Function to parse trace ID and store it if valid
static __always_inline void check_and_store_ip_trace_id(struct __ctx_buff *ctx, __u8 ip_opt_type_value) {
    __s64 trace_id;
    trace_id = trace_id_from_ctx(ctx, ip_opt_type_value); // do parsing

    // check if the trace ID is valid
    if (trace_id > 0) {
        bpf_trace_id_set(trace_id);
    } else {
        bpf_trace_id_set(0);
    }
}
```
At ingress, the function `trace_id_from_ctx` is called to parse the IP options from incoming packets. If the packet is valid, it proceeds to parse the IP options to extract the trace ID.

We loop over the IP options to extract the trace ID. This loop is unrolled to a maximum of three iterations to minimize performance impact and meet BPF requirements. The maximum of three iterations accounts for any combination of trace ID, user-provided options, and either egress NAT or DSR options. If we read an IP option from the trace type which we configured (see Section 2), then this is stored as the `ip_trace_id`.

Several edge cases are considered during parsing:
- **No IP Options:** If the packet header indicates there are no options, parsing stops early, avoiding unnecessary processing.
- **Invalid `ip_trace_id`:** invalid IDs are handled by setting the trace ID to 0.

## Section 4: `ip_trace_id` storing

Creating a per-CPU array-map allows for storage of each packet’s `ip_trace_id` as it is processed. When the feature gate is disabled (see Section 2: Feature enablement), the map is not rendered, avoiding space inefficiencies.

```c
#ifdef ENABLE_PACKET_IP_TRACING 
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, 1);
    __type(key, __u32); // only one key
    __type(value, __u64); // ip_trace_id type
    __uint(pinning, LIBBPF_PIN_BY_NAME);
} ip_trace_id_map __section_maps_btf;
...
// remaining declarations 
#endif
```

**Pros**
- Non-competitive storage.

**Considerations**
- Must ensure that data is cleared correctly.
- This is universal storage and not specifically stored in the SKB struct, so it must be cleared when a new packet is processed.
- At an ingress point, if there is no trace ID stored in a packet, the stored `ip_trace_id` is cleared from the array-map.

## Section 5: Event integration

Include ID as event field for trace and drop structs and the flow message (for Hubble integration):

```c
struct trace_notify {
    // Existing struct members
    ...
    __u64      ip_trace_id; // new trace ID field
    // Remaining struct members
    ...
};
```
```c
struct drop_notify {
    // Existing struct members
    ...
    __u64      ip_trace_id; // new trace ID field
    // Remaining struct members
    ...
};
```

```go
type Flow struct {
    // Existing struct members
    ...
    IpTraceID uint64 
    `protobuf:"varint,38,opt,name=ip_trace_id,json=ipTraceId,proto3" json:"ip_trace_id,omitempty"` // new trace ID field
    // Remaining struct members
    ...
}
```

## Benchmark Testing

### Setup

We set up our cluster with control-plane and worker nodes hosting a server and client pod, respectively. From the client pod, we ping the server pod IP address.

![CFP Demo](./images/CFP-33716-demo.png)

### Cluster setup

**Provision a Kind cluster:**
```sh
make kind && make kind-image
```
### Install Cilium

```sh
helm upgrade -i cilium ./install/kubernetes/cilium \
  --wait \
  --namespace kube-system \
  --set image.override="localhost:5000/cilium/cilium-dev:local" \
  --set image.pullPolicy=Never \
  --set operator.image.override="localhost:5000/cilium/operator-generic:local" \
  --set operator.image.pullPolicy=Never
```

### Client pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: client
  labels:
    app: client
spec:
  containers:
  - name: client
    image: busybox
    command:
    - sleep
    args:
    - "999999"
```

### Server pod YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server
  labels:
    app: server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: server
  template:
    metadata:
      labels:
        app: server
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: client
            topologyKey: kubernetes.io/hostname
      containers:
      - name: server
        image: nginx
        ports:
        - containerPort: 80
```

### Pinging

We find which node the client pod belongs to:
```sh
# kubectl get pods -o wide | grep "client"
NAME     READY   STATUS    RESTARTS   AGE    IP           NODE
client   1/1     Running   0          127m   10.244.1.97  kind-worker2
```

To find the target IP, find the IP address of the server pod:
```sh
# kubectl get pods -o wide | grep "server"
NAME                      READY   STATUS    RESTARTS   AGE    IP
server-69cdf4d5cd-mkklc   1/1     Running   0          127m   10.244.0.221
```
Now, we SSH into the worker node where the client pod is:
```sh
docker exec -it kind-worker2 bash
```
From within the worker node, we enter the client pod namespace. Then, we use the tool `nping` to generate TCP traffic, sending 1000 packets to the target IP with and without custom IP options. Use the following bash script within the node to enter the client namespace and generate traffic (with and without IP options) by pinging the server pod:

```bash
#!/bin/bash

# Usage: ./benchmark_packet_tracing.sh <target_ip> <num_with_options> <num_without_options>
# Example: sudo ./benchmark_packet_tracing.sh 192.168.1.1 100 100

TARGET_IP=$1
NUM_WITH_OPTIONS=$2
NUM_WITHOUT_OPTIONS=$3

# Validate input
if [ -z "$TARGET_IP" ] || [ -z "$NUM_WITH_OPTIONS" ] || [ -z "$NUM_WITHOUT_OPTIONS" ]; then
  echo "Usage: $0 <target_ip> <num_with_options> <num_without_options>"
  exit 1
fi

# Get container and pid
container=$(crictl ps | grep "$(crictl pods | grep client | awk '{ print $1 }')" | awk '{ print $1 }')
pid=$(crictl inspect "${container}" | grep pid | head -1 | grep -Eo "[0-9]+")

if [ -z "$pid" ]; then
  echo "Failed to get the PID of the container."
  exit 1
fi

echo "Switching to the network namespace of the container with PID ${pid}"

# Run the rest of the script in the container's network namespace
nsenter -t "${pid}" -n /bin/bash << EOF

# Function to send packets with IP options
send_packets_with_options() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] Sending $NUM_WITH_OPTIONS packets WITH IP options trace_id"
  nping --tcp -p 80 --ip-options='\x88\x04\x34\x21' -c "$NUM_WITH_OPTIONS" --rate 1000 --debug "$TARGET_IP" >> nping_output_with_options.log 2>&1
}

# Function to send packets without IP options
send_packets_without_options() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] Sending $NUM_WITHOUT_OPTIONS packets WITHOUT IP options trace_id"
  nping --tcp -p 80 -c "$NUM_WITHOUT_OPTIONS" --rate 1000 --debug "$TARGET_IP" >> nping_output_without_options.log 2>&1
}

# Start traffic generation
echo "[$(date +'%Y-%m-%d %H:%M:%S')] Starting traffic generation to $TARGET_IP"
send_packets_with_options
send_packets_without_options

echo "[$(date +'%Y-%m-%d %H:%M:%S')] Traffic generation completed."
EOF
```

In our `nping` command, `--rate 1000` sets the packet sending rate to 1000 packets per second, and `--debug` enables detailed logging. By default, Nmap uses parallelism parameters (see [here](https://nmap.org/book/man-performance.html)) determined by its own adaptive algorithms, and we do not modify those settings.

## Results

The `nping` output shows the time taken to generate, send, and wait for responses of 1000 packets under four different scenarios:
1. Baseline (with tracing feature disabled, without IP Options)
2. Test (with tracing feature disabled, with IP Options)
3. Baseline (with tracing feature enabled, without IP Options)
4. Test (with tracing feature enabled, with IP Options)

The output is logged, showing the time taken to generate, send, and wait for responses of 1000 packets.

### Baseline (with tracing feature disabled):

```sh
Baseline (without IP Options):
- Max rtt: 0.726ms | Min rtt: 0.046ms | Avg rtt: 0.046ms
- Raw packets sent: 1000 (40.000KB) | Rcvd: 1000 (44.000KB) | Lost: 0 (0.00%)
- Tx time: 1.34090s | Tx bytes/s: 29830.69 | Tx pkts/s: 745.77
- Rx time: 1.34096s | Rx bytes/s: 32812.26 | Rx pkts/s: 745.73
- Nping done: 1 IP address pinged in 1.45 seconds

Test (with IP Options):
- Max rtt: 0.236ms | Min rtt: 0.046ms | Avg rtt: 0.046ms
- Raw packets sent: 1000 (44.000KB) | Rcvd: 1000 (44.000KB) | Lost: 0 (0.00%)
- Tx time: 1.34734s | Tx bytes/s: 32657.01 | Tx pkts/s: 742.20
- Rx time: 1.34779s | Rx bytes/s: 32646.04 | Rx pkts/s: 741.96
- Nping done: 1 IP address pinged in 1.45 seconds
```

## With tracing feature enabled:

```sh
Baseline (without IP Options):
- Max rtt: 0.872ms | Min rtt: 0.046ms | Avg rtt: 0.046ms
- Raw packets sent: 1000 (40.000KB) | Rcvd: 1000 (44.000KB) | Lost: 0 (0.00%)
- Tx time: 1.33641s | Tx bytes/s: 29931.00 | Tx pkts/s: 748.28
- Rx time: 1.33649s | Rx bytes/s: 32922.03 | Rx pkts/s: 748.23
- Nping done: 1 IP address pinged in 1.45 seconds

Test (with IP Options):
- Max rtt: 0.719ms | Min rtt: 0.048ms | Avg rtt: 0.048ms
- Raw packets sent: 1000 (44.000KB) | Rcvd: 1000 (44.000KB) | Lost: 0 (0.00%)
- Tx time: 1.34727s | Tx bytes/s: 32658.61 | Tx pkts/s: 742.24
- Rx time: 1.34733s | Rx bytes/s: 32657.11 | Rx pkts/s: 742.21
- Nping done: 1 IP address pinged in 1.46 seconds
```

The additional delay introduced by enabling the tracing feature is relatively small. When the feature is disabled, the average RTT (round-trip time) remains the same (0.046ms).

When the tracing feature is enabled, the average RTT increases by 0.002ms when IP Options are used (0.046ms vs 0.048ms). Overall, the performance impact of enabling the tracing feature and using IP Options is minimal, making it a feasible solution for environments requiring detailed packet tracing without significant degradation in network performance. Moreover, as this is a debugging feature, in environments where this feature is disabled, there is no performance loss.

## Future Milestones

### Deferred Milestone 1: Use conntrack to restore trace ID for reply packets
- When a packet includes a trace ID, the conntrack entry should record the ID along with the connection information.
- For subsequent packets – if a trace ID is not present in the IP Options – use the track ID from conntrack to restore the internal state.
- Note this will not modify the IP option header.

### Deferred Milestone 2: Tag packets
- **Audience:** Proficient Cilium user with knowledge of cilium-dbg / pwru tools.
- Provide a `cilium-dbg` subcommand for a user to define what traffic should be traced.
- Prepend tag program to container lxc interface.
- **Optional:** Provide a flag to optionally remove the tag before egressing Node NIC, addressing targets that do not support IP options.
- **Challenge:** This affects the available MTU.
  - [Preferred] Only apply to packets that have the length available (e.g., packets with no data; SYN w/o fast open, RST, etc.).
  - Modify the MTU/route MTU of the container (i.e., subtracting the IP option + trace ID lengths).

### Deferred Milestone 3: Tag packets via CR
- **Audience:** Network specialist with knowledge of network topology.
- Develop a K8s API where the user specifies traffic to tag using common selectors: 4-tuple, pod identity, etc.

## Appendix

### Parsing attempted approaches

#### [Nonviable] Option 2: Parse at XDP hook
**Pros:**
- Earliest detection of trace IDs.
- Immediate handling of packets with trace IDs.

**Considerations:**
- XDP programs don’t support the SKB struct.
- Must ensure that TC hook is able to parse from the xdp_md struct created in the XDP hook, leading to increased complexity.

### Trace ID storage attempted approaches

#### [Nonviable] Option 2: Use sk_buff extensions
**Pros:**
- Can extend the sk_buff structure without modifying the core structure.
- Data freed when skb is freed.

**Considerations:**
- Compatible with kernel versions post-v5.0-rc1.

#### [Nonviable] Option 3.1: Use skb->mark or skb->CB field
**Pros:**
- The “mark” field is a generic packet mark for sk_buff and suitable for such purposes as adding a trace ID.

**Considerations:**
- Potential conflicts with other parts of the network that use this field.

#### [Nonviable] Option 3.2: Store reference into skb->mark or skb->CB field
**Pros:**
- Minimal increase in skb storage.
- For packets that don’t have trace IDs, we don’t modify the sk_buff struct.

**Considerations:**
- Accessing the data requires extra memory access (dereferencing the pointer).
- Management of two buffers concurrently introduces additional complexity.

