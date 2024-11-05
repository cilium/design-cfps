**CFP-35631: Support CNI STATUS operation in Cilium-CNI**

# **Meta**

**SIG: SIG-DATAPATH**  

**Sharing: Public**  

**Begin Design Discussion:** 2024-10-30  

**Cilium Release:** 1.18  

**Authors:** Praveen Krishna <pkrishn@google.com>, Archit Kulkarni <architkulkarni@google.com>

**Status:** Implementable

## **Summary**

Add support for the [CNI STATUS operation](https://github.com/containernetworking/CNI/blob/main/SPEC.md#status-check-plugin-status) in cilium-cni. This is to allow container runtimes to determine the network readiness of the node.  cilium-cni must signal its readiness to handle CNI ADD requests by returning a zero exit code. Returning a non-zero code (accompanied by an error message on standard output) indicates that the plugin cannot process these requests. 

## **Motivation**

General motivation for the introduction of CNI STATUS operation can be found here: [Proposal: "STATUS" verb](https://github.com/containernetworking/cni/issues/859)

This CFP proposes that cilium-cni should add support for the STATUS verb which provides a more detailed way to pass the health of cilium-cni and cilium-agent back to container runtime.

Below is how kubelet determines if a Node is “Network Ready”

* The  CNI configuration file defines how we set up the network for a pod. When a container runtime needs to set up networking for a container, it looks at this file and exec’s the CNI plugin binaries in the config.  
* The [NetworkReady](https://github.com/kubernetes/kubernetes/blob/34abd36e88e50d913af9413f9353515deb7f09d4/pkg/kubelet/kubelet.go#L2873-L2876) condition in kubelet is used to determine if the pods can be scheduled on the node and is initialized to false when the node starts. Kubelet calls the [CRI Status() API](https://github.com/kubernetes/kubernetes/blob/3718932d9a3ea459217ca11efe1ae9ead6aa92c1/pkg/kubelet/kubelet.go#L2871) periodically and sets this flag to true when the container runtime returns the NetworkReady status. Kubelet continues to use the same API to monitor the node readiness.  
* The container runtime uses the presence of a CNI configuration file as an indicator of the CNI plugin's readiness to process ADD requests. This inferred status is then reported to kubelet.

Checking for a CNI config file is an unreliable indicator of network health on the node, as various issues can prevent proper network functionality despite the file's existence. Some potential issues include

* cilium-agent cannot be scheduled during upgrade (due to resource constraints on the node).  
* cilium-agent is crash looping.  
* CNI config file is invalid.  
* We run out of IP addresses on the node.

There is a parallel effort in progress in go-cni ([https://github.com/containerd/go-cni/pull/121](https://github.com/containerd/go-cni/pull/121)) and CRI to call CNI STATUS and pass the status to kubelet. This would enable kubelet to set NetworkReady to false and prevent workload pods from being added (KEP is WIP).

## **Goals**

* Implement STATUS in Cilium with the minimal return code 50\.  [“50: The plugin is not available (i.e. cannot service ADD requests)”](https://github.com/containernetworking/CNI/blob/main/SPEC.md#status-check-plugin-status)

## **Non-Goals**

* Define “limited connectivity” and implement return code 51\.  [“51: The plugin is not available, and existing containers in the network may have limited connectivity.”](https://github.com/containernetworking/CNI/blob/main/SPEC.md#status-check-plugin-status)  
* Integration work with kubelet, containerd, go-cni

## **Proposal**

1. Add the STATUS API in the [ChainingPlugin interface](https://github.com/cilium/cilium/blob/78e8f73bace4ed024a8781325fa4882efaa1e026/plugins/cilium-cni/chaining/api/api.go#L42) and supported [CNI functions](https://github.com/cilium/cilium/blob/0bc32ae1101e3fd0e592e35353cf835b1ddaa584/plugins/cilium-cni/cmd/cmd.go#L94) in cilium-cni. Update the supported [CNI spec version](https://github.com/cilium/cilium/blob/0bc32ae1101e3fd0e592e35353cf835b1ddaa584/plugins/cilium-cni/main.go#L23) to 1.1.0.   
2. Check cilium-cni connectivity with cilium-agent using [healthz endpoint](https://github.com/cilium/cilium/issues/27243).  
3. If cilium-cni depends on [delegated plugins](https://github.com/cilium/cilium/blob/78e8f73bace4ed024a8781325fa4882efaa1e026/pkg/ipam/option/option.go#L35) (host-local, azure etc) for IPAM, cilium-cni invokes the STATUS API of the delegated plugins and pass the result back.

## **Future Milestones**

1. Add an endpoint in cilium-agent that allows for the detection of errors with IPAM (such as IP exhaustion) when delegated plugins are not used.   
2. Check if pod interfaces can be created successfully.  
3. Add a more detailed health check and include things such as cilium-agents connectivity with the API server ([https://github.com/cilium/cilium/issues/28390](https://github.com/cilium/cilium/issues/28390))  etc. 
