# CFP-31904: CiliumEndpointSlice Graduation to Stable

**SIG: sig-scalability**

**Begin Design Discussion:** 2024-04-11

**Cilium Release:** 1.16

**Authors:** Tim Horner <timothy.horner@isovalent.com>, Marcel Zięba <marcel.zieba@isovalent.com>, Marco Iorio <marco.iorio@isovalent.com>

## Summary

CiliumEndpointSlice facilitates the grouping of multiple CiliumEndpoint objects to achieve better scalability. This document describes the remaining work needed to consider CiliumEndpointSlice a Stable feature.

## Goals

* Blanket test CiliumEndpointSlice in all CI workflows.
    - CiliumEndpointSlice will be temporarily enabled for all CI workflows to identify failures or feature incompatibilities. The goal of the blanket test is to provide guidance on areas of improvement to focus on and any gaps in tooling.

* Enable CiliumEndpointSlice in CI tests.
    - The feature has unit test coverage but there is no end-to-end testing. CiliumEndpointSlice will be enabled in the Conformance E2E and 100 Nodes Scale Test workflows. Additional workflows may be added based on the findings of the blanket test.

* Identify feature incompatibilites
    - Identify features that rely on the use of CiliumEndpoint and summarize the work needed to support CiliumEndpointSlice. A decision should be made per-feature whether the incompatibility should be fixed or documented. The decision should be made based on the amount of work needed and the value add of making the feature compatible with CiliumEndpointSlice.
* Simplify CiliumEndpointSlice configuration
    - Cilium Operator accepts the following 8 options for configuring CiliumEndpointSlice:
        - ces-max-ciliumendpoints-per-ces
	    - ces-slice-mode
	    - ces-write-qps-limit
	    - ces-write-qps-burst
	    - ces-enable-dynamic-rate-limit
	    - ces-dynamic-rate-limit-nodes
	    - ces-dynamic-rate-limit-qps-limit
	    - ces-dynamic-rate-limit-qps-burst

    - Configuration should be simplified by setting sane defaults and removing any unused or deprecated options. CiliumEndpointSlice should use its own Kubernetes ClientSet to decouple it from the ClientSet used by the CiliumOperator for all other resources.

* Define a migration path from CiliumEndpoint to CiliumEndpointSlice
    - A migration path is needed to allow users to enable CiliumEndpointSlice on clusters utilizing CiliumEndpoint. The process should be simple and well documented, with CI test coverage to identify regressions. 