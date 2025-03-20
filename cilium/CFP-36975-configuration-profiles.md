# Cilium Feature Proposal: Configuration Profiles

## Author(s)

Dorde Lapcevic (dordel@google.com)

## Date

2025-02-07

## Status

Implementable

## Abstract

This proposal introduces the concept of "Configuration Profiles" to Cilium.  Profiles are pre-defined, documented, and tested sets of Cilium configuration options designed to address specific operational needs and use cases.  This aims to simplify deployment, improve predictability, and enhance the user experience by providing well-defined operational modes.

## Motivation

Currently, Cilium offers numerous feature flags that operators configure individually.  This granular control, while powerful, presents several challenges:

*   **Combinatorial Complexity:** The interaction between numerous flags can lead to unexpected behavior and makes exhaustive testing difficult.  Certain combinations may be subtly broken or have performance implications that are not immediately apparent.
*   **Operational Expertise Required:**  Operators need a deep understanding of individual flags and their interplay to configure Cilium optimally for their environment.
*   **Lack of Clear Guidance:**  New users may struggle to understand the best configuration for their needs, leading to suboptimal deployments.
*   **Implicit Knowledge:**  Experienced operators often develop a set of preferred flags based on their specific environment (scale, performance requirements, network topology, etc.). This knowledge is often implicit and not easily shared.

Configuration Profiles address these issues by:

*   **Simplifying Deployment:**  Profiles provide a "one-click" (or "one-command") deployment option for common use cases.
*   **Improving Predictability:**  Each profile will be thoroughly tested and documented, ensuring consistent behavior and performance.
*   **Providing Clear Guidance:** Documentation will clearly explain the purpose, benefits, limitations, and enabled/disabled features of each profile.
*   **Best Practices:** Profiles encapsulate expert knowledge and best practices for specific operational scenarios.
*   **Targeted Testing:** By testing at the profile level (a set of features) instead of only individual flags, we increase coverage and reduce the risk of unexpected interactions.

## Goals

The main goals of this proposal are:
*   Define a process for creating a new Configuration Profile
*   Establish a testing framework to ensure the stability and correctness of each profile.
*   Enable the community to propose and contribute new profiles, including updating which features work with which profile.

## Non-Goals

*   Completely eliminate individual feature flags.  Advanced users will still be able to customize their deployments beyond the pre-defined profiles.  Profiles are intended to be a starting point, not a restriction.
*   Create a profile for every possible combination of flags.  The focus is on common, well-understood use cases.
*   Guarantee that profiles will cover 100% of user needs. It's recognized there will always be edge cases, but this helps a majority of users.

## Example Profile

*   **High-Scale:**  Optimized for large-scale clusters, with high number (thousands) of nodes and high pod churn rate (hundreds per second). Limited to a set of basic networking and K8s features: pod connectivity, K8s Service and basic observability.
    *   `--enable-policy=never`
    *   `--enable-k8s-networkpolicy=false`
    *   `--enable-cilium-network-policy=false`
    *   `--enable-cilium-clusterwide-network-policy=false`
    *   `--identity-allocation-mode=crd`
    *   `--disable-endpoint-crd=true`
    *   `--enable-cilium-endpoint-slice=false`

    Note: This is only an example profile. It's not ready for use.

## Creating a Profile

*   Provide clear documentation for each profile, including:
    *   Purpose and use case
    *   Benefits and limitations (trade-offs)
    *   Specific configuration options (flags) enabled/disabled
    *   Installation instructions
    *   Supported Cilium versions
    *   Known issues
*   Implement profile selection via Helm charts.

## Implementation

The implementation will leverage Cilium's Helm chart capabilities:

1.  **Helm Chart Values:**  Each profile will be defined as a set of Helm values overrides.  These overrides will configure the necessary Cilium feature flags.  For example, the `values.yaml` for the "High-Scale" profile might include:

    ```yaml
    # profiles/high-scale.yaml
    disableEndpointCRD: false
    policyEnforcementMode: "never"
    # ... other relevant settings ...
    ```

2.  **Profile Selection:** Users will select a profile during installation by referencing the appropriate values file:

    ```bash
    helm install cilium cilium/cilium --version 1.17.x \
      --namespace kube-system \
      --values profiles/high-scale.yaml
    ```

3.  **Documentation:** Each profile will have a dedicated page in the Cilium documentation, detailing its characteristics, installation instructions, incompatible features and functionality, and relevant configuration options.  This documentation will be part of the Cilium documentation repository.

4. **Testing:**
    *   **Unit Tests:** The Helm chart rendering will be unit-tested to ensure that the correct flags are set for each profile.
    *   **Integration/E2E Tests:**  A new test suite will be created to validate the functionality and stability of each profile. These tests will run against a Kubernetes cluster with Cilium installed using the profile's Helm values. A subset of existing tests will run together with new per-profile tests that cover key features and use cases relevant to the profile.
    *   **Continuous Integration:** Profile tests will be integrated into the Cilium CI pipeline to ensure that changes to Cilium do not break existing profiles.

## Alternatives Considered

*   **Individual Flags:**  As discussed in the "Motivation" section, this approach is complex and error-prone.
*   **Separate Helm Charts:**  Creating separate Helm charts for each profile would lead to significant code duplication and maintenance overhead.

## Open Issues

*   Onboard initial profiles. This requires community input.
*   Determine the best way to handle profile updates and versioning.  How do we ensure that users can safely upgrade Cilium while using a specific profile?
    *   **Couple Config to Cilium Releases** (Suggested during the review): Each Cilium release will have a set of profiles that work specifically on that Cilium release (e.g. Cilium 1.17 has a Profile-X-1-17 and Profile-Y-1-17 that are tailored for the Cilium 1.17 version).
*   Develop detailed test plans for each profile.
*   Consider how to allow users to extend profiles (create custom profiles based on existing ones) without modifying the core Helm charts.

## Next Steps

Proceed with implementing one of the initial candidate profiles.
* Initial Documentation.
* Create the necessary Helm chart values file.
* Create a test suite.
