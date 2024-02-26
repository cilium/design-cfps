# CFP-2023-10-12: Enhanced HTTPS Observability: Capturing Server Name Indication (SNI) without Traffic Decryption


## Meta

**SIG: SIG-Observability**

**Sharing: Public**

**Begin Design Discussion:** October 12, 2023

**End Design Discussion:** -

**Cilium Release:** -

**Authors:** myusefpur@gmail.com


## Summary

_Introduce a feature to capture and observe server names (SNI) from HTTPS/TLS traffic without needing to decrypt the entire session. This would enhance network visibility and troubleshooting without compromising data privacy._


## Motivation

_Modern network observability, particularly in Kubernetes, demands a granular insight into traffic. Cilium's current capability offers visibility into L7 HTTP traffic but lacks the same depth for encrypted HTTPS traffic. Understanding server interactions in HTTPS is essential for security, traffic management, and troubleshooting, hence the need for this feature._


## Goals

* _Extract SNI information from HTTPS/TLS traffic without decryption._
* _Integrate the feature seamlessly into Cilium's existing observability framework._
* _Ensure that the implementation has minimal overhead on traffic performance._
* _Provide user-friendly methods (annotations) for users to activate SNI capture_


## Non-Goals

* _Full traffic decryption or TLS termination._
* _Extraction of any other encrypted data beyond SNI._


## Proposal


#### Overview

This feature proposes to use eBPF programs to intercept and analyze TLS handshake packets for extracting SNI information. The extracted SNI would then be integrated into Cilium's observability tools for monitoring and logging.


#### Section 1: eBPF-based SNI Extraction

* **Working**:
    * Intercept TLS handshake packets.
    * Parse packets to extract the SNI.
    * Export the data for integration with monitoring/logging tools.


#### **Section 2: User Control and Integration**

* **Annotations**:
    * Extend `policy.cilium.io/proxy-visibility` for enabling SNI extraction.
    * Grant users the ability to specify which services/pods require SNI extraction.
* **Integration**:
    * Export extracted SNI data to hubble


# Impacts / Key Questions

**Impact 1**: Performance Overhead

* Capturing SNI might introduce slight performance overhead. However, with eBPF's efficiency, this impact should be minimal.

**Key Question 1**: Safety of Data Extraction

* How do we ensure that only the SNI is extracted, and no other sensitive data is unintentionally accessed?

**Option 1: Strict eBPF Filtering**

* **Pros**:
    * Targeted extraction, minimizing risk.
    * Efficient and fast.
* **Cons**:
    * Requires meticulous coding and testing to ensure accuracy.

**Option 2: Post-Extraction Data Sanitization**

* **Pros**:
    * Adds an extra layer of security.
    * Easier to implement than strict filtering.
* **Cons**:
    * Introduces additional latency.


# Future Milestones

**Deferred Milestone 1: Enhanced Decryption Capabilities**

* Introduce methods for users to optionally provide TLS certs to decrypt traffic for deeper inspection, going beyond just SNI extraction.

**Deferred Milestone 2: Advanced Anomaly Detection**

* Utilize the SNI data for anomaly detection, highlighting any unauthorized or unexpected domain interactions.
