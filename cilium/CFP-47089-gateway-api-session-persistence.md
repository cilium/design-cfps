# CFP-47089: Gateway API Session Persistence

## Meta

SIG: SIG-ServiceMesh
Sharing: Public
Begin Design Discussion: July 10, 2026
End Design Discussion: TBD
Cilium Release: 1.20
Authors: tim <thorn3r@cisco.com>

## Summary

This proposal extends Cilium's Gateway API implementation to support session
persistence via the Route Rule API as defined in
[GEP-1619](https://gateway-api.sigs.k8s.io/geps/gep-1619/#achieving-session-persistence).
Session persistence is part of Gateway API's experimental release channel and
is subject to change. However, the scope of this proposal is deliberately narrow
and focuses on delivering the core features required by Gateway API that are
widely adopted by other implementations. Cilium will support cookie-based
persistence on `HTTPRoute` and `GRPCRoute` rules with session-based lifetime.

## Motivation

Some applications require requests in a client session to continue reaching the
same backend endpoint. Gateway API exposes this through `sessionPersistence` on
route rules. Cilium already translates Gateway API routes into an internal HTTP
model and Envoy xDS resources, but currently ignores route-rule session
persistence.

## Goals

- Support `sessionPersistence` on `HTTPRoute` and `GRPCRoute` rules.
- Support cookie-based persistence.
- Support session-based lifetime for cookies.
- Reject unsupported session persistence fields with route status.

## Non-Goals

- `BackendTrafficPolicy` session persistence.
- Header-based persistence.
- Permanent cookies or `absoluteTimeout`.
- Configurable cookie security attributes.
- Signed or encrypted upstream-host cookies.

## Proposal

### Supported API Surface

The following fields are supported on `HTTPRoute` and
`GRPCRoute` rules:

```yaml
sessionPersistence:
  type: Cookie
  sessionName: gateway-session
  cookieConfig:
    lifetimeType: Session
```

Unsupported fields:

- `sessionPersistence.absoluteTimeout`

Unsupported values:

- `sessionPersistence.type: Header`
- `sessionPersistence.cookieConfig.lifetimeType: Permanent`

Routes using unsupported fields or values receive `Accepted=False` with reason
`UnsupportedValue`.

### Rendering and Status

Cilium will translate supported route-rule persistence into
`envoy.filters.http.stateful_session` with per-route
`envoy.http.stateful_session.cookie` configuration. Since the only supported
lifetime type is `Session`, Cilium will set the Envoy cookie `ttl` to `0s`,
causing Envoy to generate a session cookie rather than a persistent cookie.
Cilium will set `Secure`, `HttpOnly`, and `SameSite=Strict`, as suggested by
upstream documentation.

The stateful session configuration will use non-strict behavior. If the endpoint
encoded in the cookie is unavailable, Envoy will fall back to normal load
balancing instead of failing the request.

Because Envoy HTTP filters are configured at listener scope, Cilium will
explicitly disable the stateful-session filter on routes that do not use session
persistence, including redirect and direct-response routes that do not select an
upstream backend.

During ingestion, session persistence fields are validated and stored in the
internal model. If the route contains unsupported fields or values, the route
status will be updated to indicate that it was rejected. Session persistence
settings are included in the route match key so routes with the same match but
different persistence settings are not merged. Cilium derives the cookie path
from exact or prefix path matches, falling back to "/" if the path can't be
determined.

```
type HTTPSessionPersistence struct {
 Cookie *HTTPCookieSessionPersistence `json:"cookie,omitempty"`
}

type HTTPCookieSessionPersistence struct {
 Name     string `json:"name,omitempty"`
 Path     string `json:"path,omitempty"`
 Secure   bool   `json:"secure,omitempty"`
 HTTPOnly bool   `json:"httpOnly,omitempty"`
 SameSite string `json:"sameSite,omitempty"`
}
```

## Key Questions

### Key question #1: Should Cilium support policy attachment?

Policy attachment through the `BackendTrafficPolicy` API introduces additional
complexity in precedence and conflict handling and does not seem widely
supported in other Gateway API implementations.

Recommendation: Support route rules only for this phase of development.

### Key question #2: Should Cilium make the upstream host cookie tamper-proof?

Envoy's session state stores the upstream host address in the cookie. Making the
cookie tamper-proof would likely require upstream work in Envoy to support
signing or encryption, alongside key management. This is out of scope for the
current proposal and can be revisited as a follow-up.

## Future Milestones

- Support header persistence.
- Support permanent cookies and `absoluteTimeout`.
- Support configurable cookie attributes if the secure defaults are too
  restrictive for real deployments.
- Evaluate signed or encrypted upstream-host cookies.
