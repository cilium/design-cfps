# CFP-48102: Cilium Gateway API TLS Options
 
**SIG:** SIG-ServiceMesh
 
**Begin Design Discussion:** 2026-08-21
 
**Cilium Release:** 1.21
 
**Authors:** Aleksandr Rybolovlev <aleksandr.rybolovlev@proton.me>
 
**Status:** Draft
 
## Summary
 
This proposal adds Cilium-specific Gateway API TLS options for listeners with downstream TLS termination.
 
The options can be used on both `Gateway` and `ListenerSet` listeners to configure TLS protocol versions, cipher suites, ECDH curves, and signature algorithms.
 
## Motivation
 
Today, Cilium relies on Envoy defaults for downstream TLS parameters on Gateway API listeners. Envoy defaults are reasonable for many deployments, but they are not always enough when operators need explicit control over TLS policy.
 
For example, an operator may require TLS 1.3 only, or allow TLS 1.2 only for compatibility.
 
Gateway API provides an implementation-specific listener extension point through `spec.listeners[].tls.options`, but Cilium does not currently consume these options. As a result, users cannot configure listener-specific downstream TLS protocol versions or related TLS parameters through Gateway API listener configuration.
 
## Goals
 
- Add Cilium-specific TLS options under listener `tls.options` for `Gateway` and `ListenerSet` resources.
- Support configuring the minimum and maximum downstream TLS protocol versions.
- Support configuring TLS cipher suites for protocol versions where Envoy applies cipher suite configuration.
- Support configuring ECDH curves and signature algorithms for advanced TLS tuning.
- Validate Cilium TLS options and reject invalid listener TLS configurations.
- Map the supported Gateway API TLS options to Envoy `TlsParameters`.
 
## Non-Goals
 
- Introduce a new Cilium CRD or attached policy for listener TLS settings.
- Change TLS certificate handling, `certificateRefs`, SDS secret loading, or secret synchronization.
- Change TLS passthrough behavior. These options apply only when Cilium terminates downstream TLS.
- Configure backend TLS origination or `BackendTLSPolicy`.
- Configure downstream client certificate validation or mutual TLS.
- Add support for TLS 1.0 or TLS 1.1 in the initial implementation.
 
## Proposal
 
### Overview
 
Cilium will read Cilium-specific TLS options from listener `tls.options` on both `Gateway` and `ListenerSet` resources for listeners that terminate downstream TLS.
 
The implementation will translate supported options into Envoy `TlsParameters` on the listener `DownstreamTlsContext`.
 
If a listener does not set any supported Cilium TLS options, Cilium will keep the current behavior and rely on Envoy defaults.
 
### Supported options
 
The implementation will support the following keys:
 
```yaml
cilium.io/tls-min-version: "<TLS_VERSION>"
cilium.io/tls-max-version: "<TLS_VERSION>"
cilium.io/tls-cipher-suites: "<CIPHER_SUITES_LIST>"
cilium.io/tls-ecdh-curves: "<ECDH_CURVES_LIST>"
cilium.io/tls-signature-algorithms: "<SIGNATURE_ALGORITHMS_LIST>"
```
 
`cilium.io/tls-min-version` and `cilium.io/tls-max-version` configure the downstream TLS protocol version range. The initial implementation will support TLS 1.2 and TLS 1.3. Accepted user-facing values are `"1.2"` and `"1.3"`.
 
If either option is not set, Cilium will leave the corresponding Envoy field unset and Envoy will use its default for that bound.
 
When the minimum and maximum versions are equal, Cilium will allow only that TLS version.
 
```yaml
tls:
  options:
    cilium.io/tls-min-version: "1.3"
    cilium.io/tls-max-version: "1.3"
```
 
`cilium.io/tls-cipher-suites` configures a comma-separated Envoy TLS cipher suite list for TLS 1.2 handshakes. It does not configure TLS 1.3 cipher suites.
 
Cilium will pass valid cipher suite values through to Envoy even when the configured TLS version range is TLS 1.3 only; in that case, the option is accepted but has no effect on TLS negotiation.
 
```yaml
cilium.io/tls-cipher-suites: "ECDHE-RSA-AES128-GCM-SHA256,ECDHE-RSA-AES256-GCM-SHA384"
```
 
`cilium.io/tls-ecdh-curves` and `cilium.io/tls-signature-algorithms` configure comma-separated lists for advanced TLS tuning.
 
These options provide the TLS 1.3-relevant tuning surface because TLS 1.3 separates supported groups and signature algorithms from cipher suites. They can also affect TLS 1.2 handshakes.
 
```yaml
cilium.io/tls-ecdh-curves: "X25519,P-256"
cilium.io/tls-signature-algorithms: "ecdsa_secp256r1_sha256,rsa_pss_rsae_sha256"
```
 
If the user does not specify these list options, Envoy will use its defaults.
 
### Validation
 
Cilium will validate supported Cilium TLS options before generating Envoy configuration.
 
Cilium validation is API-level and does not attempt full runtime compatibility validation.
 
Cilium will reject unsupported `cilium.io/*` TLS option keys, unsupported TLS versions, invalid version ordering, listeners where Cilium does not terminate downstream TLS, and TLS parameter values that are not in the supported value sets.
 
Failures that depend on the certificate, Envoy build, or runtime environment are left to Envoy and surfaced through the existing listener programming error path.
 
Cilium will reject configurations where:
 
- A supported option is configured on a listener where Cilium does not terminate downstream TLS.
- A TLS version is not supported by the implementation.
- The minimum TLS version is greater than the maximum TLS version.
- A list-valued option is empty or contains empty entries.
- An unsupported Cilium TLS option key is present.
 
When a supported Cilium TLS option is invalid, or when an unsupported Cilium TLS option is present, Cilium will reject only the affected listener and exclude it from the generated Envoy configuration.
 
For `Gateway` listeners, Cilium will set the listener `Accepted` condition to `False` with the standard `UnsupportedValue` reason, and the `Programmed` condition to `False` with the `Invalid` reason.
 
For `ListenerSet` listener entries, Gateway API does not define a standard `ListenerEntryStatus` `Accepted` reason for invalid implementation-specific listener options as of v1.6. Cilium will set the listener entry `Accepted` condition to `False` using an implementation-specific reason, and `Programmed` to `False` with the `Invalid` reason.
 
Options with non-`cilium.io/` domain prefixes will be ignored by this feature.
 
The condition message will identify the invalid option and explain why it was rejected.
 
### Envoy mapping
 
Cilium will map Gateway API TLS options to Envoy `TlsParameters` as follows:
 
- `cilium.io/tls-min-version` -> `tlsParams.tlsMinimumProtocolVersion`
- `cilium.io/tls-max-version` -> `tlsParams.tlsMaximumProtocolVersion`
- `cilium.io/tls-cipher-suites` -> `tlsParams.cipherSuites`
- `cilium.io/tls-ecdh-curves` -> `tlsParams.ecdhCurves`
- `cilium.io/tls-signature-algorithms` -> `tlsParams.signatureAlgorithms`
 
Cilium will translate user-facing TLS version values to the corresponding Envoy `TlsProtocol` enum values, for example `"1.2"` to `TLSv1_2` and `"1.3"` to `TLSv1_3`.
 
Comma-separated list options will be split, validated, and passed to Envoy in the configured order.
 
## Impacts / Key Questions
 
### Impact: Validation becomes part of listener programming
 
`tls.options` is a string map, so Kubernetes schema validation cannot validate option-specific values. Cilium must validate supported options during reconciliation before generating Envoy configuration.
 
Invalid Cilium TLS options will cause Cilium to reject only the affected listener. Other valid listeners on the same `Gateway` or `ListenerSet` will continue to be programmed.
 
### Key Question: How should listener TLS option failures affect parent status?
 
This CFP defines listener-level behavior: invalid listener-scoped Cilium TLS options reject only the affected listener. Other valid listeners on the same `Gateway` or `ListenerSet` continue to be programmed.
 
The implementation still needs to decide how these partial listener failures are reflected in parent `Gateway` and `ListenerSet` status conditions.
 
One option is to preserve existing aggregate parent status behavior and rely on listener conditions for the precise failure. Another option is to surface partial listener failures in parent-level conditions so operators can detect the issue without inspecting every listener.
 
## Future Milestones
 
### Support deprecated TLS versions for compatibility
 
A future CFP may add explicit support for TLS 1.0 and TLS 1.1 for legacy compatibility scenarios. This future work would require explicit cipher suite configuration and clear documentation of the security risks.
 
### Add typed TLS policy and profiles
 
A future CFP may introduce a typed Cilium policy or attached policy resource for listener TLS settings if string-based `tls.options` becomes too limited or difficult to validate.
 
Such a policy could also expose predefined TLS profiles, such as modern, intermediate, or compliance-oriented profiles, to reduce the need for users to configure individual cipher suites, curves, and signature algorithms directly.
 
### Expose Envoy TLS compliance policies
 
Envoy supports TLS compliance policies in `TlsParameters`. A future CFP may evaluate whether Cilium should expose these policies for Gateway API listeners.
 
### Extend TLS tuning to backend connections
 
A future CFP may evaluate similar TLS parameter configuration for backend TLS origination. Backend TLS origination is separate from listener downstream TLS configuration.
