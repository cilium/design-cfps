# CFP-26521: Expose URL path in Hubble HTTP metrics

**SIG: SIG-HUBBLE-API/Observability**

**Begin Design Discussion:** 2026-06-21

**Cilium Release:** TBD

**Authors:** Amaan Ul Haq Siddiqui <amaanulhaq.s@outlook.com>

**Status:** Implementable

## Summary

Add an opt-in `path` label to the Hubble HTTP Prometheus metrics. The label value comes from matching the request URL path against operator-supplied templates. The value emitted is the template string itself, so the set of possible label values is fixed by the config, not by traffic.

## Motivation

Hubble already parses the HTTP URL from every L7 flow and shows it in `hubble observe`, but the Prometheus metrics do not carry it. Per-route metrics like "which routes are slow" and "which routes return 5xx" require exporting flow logs and aggregating them somewhere else.

cilium/cilium#26521 was filed in June 2023, closed on cardinality grounds, and reopened in February 2024. Raw paths are effectively unbounded, so a raw `path` label would produce unbounded Prometheus cardinality. Templating bounds it: `/users/abc/orders` and `/users/xyz/orders` collapse into `/users/{id}/orders`.

## Goals

* Add a `path` label to the Hubble HTTP metrics (`hubble_http_requests_total`, `hubble_http_responses_total` in V1, `hubble_http_requests_total` in V2, and `hubble_http_request_duration_seconds` in both).
* Opt-in only. When the feature is not enabled, metric shapes are unchanged.
* A distinct-value count for `path` that is fixed by the config alone, deterministic across agent restarts.
* Works with either the legacy `hubble-metrics` config string or the dynamic config map from CFP-30788. The agent accepts one of the two at a time.

## Non-Goals

* Exposing query strings, fragments, or request bodies.
* Auto-learning templates from observed traffic. Operators provide the templates.
* Adding a `path` label to DNS, gRPC, or non-HTTP metrics. Same problem, separate CFP.
* Replacing the flow-export pipeline.
* Coupling the feature to L7 `CiliumNetworkPolicy`. This is discussed below as a deferred alternative.

## Proposal

### Overview

Add an optional `pathContext` block to the HTTP metric config. When set, the HTTP handler derives a `path` label by matching the request URL path against the templates. On no match, the configured miss policy fires. The label value is always a template string or the fixed miss value, never anything derived from the request, so the full set of label values is known at config load.

This happens after the L7 parser fills the flow's HTTP `Url` field. No BPF or proxy changes.

### Handler change

`pkg/hubble/metrics/http/handler.go` registers the metrics in `Init`. With `pathContext` set, `path` is appended to the label sets of:

* `hubble_http_requests_total`
* `hubble_http_responses_total` (V1 only)
* `hubble_http_request_duration_seconds`

With `pathContext` unset, no label is added. The registered metrics are identical to today.

The match input is the flow's HTTP URL, read via `flow.GetL7().GetHttp().GetUrl()`. With the Envoy proxy, this is a full URL of the form `scheme://host/path`, built from Envoy access-log fields in `pkg/envoy/accesslog.go:ParseURL`. The handler parses it with `net/url` and uses the `Path` component for matching. Query strings and fragments are not used. A URL that fails to parse counts as a miss.

V1 increments `hubble_http_requests_total` on the REQUEST flow, and `hubble_http_responses_total` plus the duration histogram on the RESPONSE flow. The matcher runs once on each. The parser fills `Url` on both, so the templated label value is the same on both sides.

V2 only processes the RESPONSE flow. The matcher runs once.

### Implementation: template matching

Small segment-based matcher. No regex engine in v1.

At config load, each template is compiled once into an ordered list of segment matchers:

* literal segment (exact string match)
* `{name}` placeholder (exactly one segment, never empty, no slashes)
* `{name...}` greedy tail (rest of the path including slashes, may be empty, must be the last segment)

The two wildcard forms are the ones Go's `net/http.ServeMux` accepts since Go 1.22, so the syntax is already familiar. A placeholder must span a whole segment (`/b_{bucket}` is invalid), as in ServeMux. Placeholder names are restricted to `[A-Za-z_][A-Za-z0-9_]*`, an ASCII-only subset of ServeMux's Go-identifier rule. ServeMux itself is not reusable as the engine: its matcher is not exported, and most-specific-wins is the wrong semantic for an ordered template list. The matcher here is simpler, and the differences are part of the spec:

* Templates match in declaration order, first match wins. There is no most-specific-wins ranking, so a broad template listed first shadows a narrower one listed later.
* A template always matches the whole path. A trailing slash is a literal character, not a subtree match, so ServeMux's `{$}` token is not needed and not supported. Subtree capture is written out as `/static/{rest...}`, which matches `/static/` (empty tail) but not `/static`.
* Segments compare byte for byte. No percent-decoding, no dot-segment normalization. The input is the path as the L7 parser recorded it.

At request time: split the URL path on `/`, walk the compiled templates in declaration order, first match wins. Matching is a linear walk over at most 100 compiled templates per metric. The label value emitted is the template string with placeholders intact, not the request path. `/api/v1/users/abc` and `/api/v1/users/xyz` both produce `path="/api/v1/users/{id}"`.

A malformed template, a duplicate template, an `onMiss.value` equal to one of the template strings, or a template list longer than the cap (see Bounding cardinality) is a config validation error. For the dynamic config map this follows the existing watcher behavior (`pkg/hubble/metrics/metric_config_watcher.go`): the reload is rejected, the last good config stays active, and the error is logged. Today the watcher only logs; the load-errors metric under Self-observability adds a counter for these rejections. For the legacy string, validation fails Hubble metrics initialization, the same as an invalid context option today.

The grammar reserves `:` inside placeholders for a future per-segment regex variant (`{name:regex}`). Adding it later is a parser extension. Existing templates keep working.

### Config

Two ways to configure it, matching how Hubble metric options already work.

Legacy flat string form for `hubble-metrics`. The parser splits on the first `:` to separate the metric name from its options, then on `;` between options, then on `|` between values inside one option (`pkg/hubble/metrics/api/api.go:ParseStaticMetricsConfig`):

```text
http:exemplars=true;sourceContext=pod;destinationContext=pod;path=/api/v1/users/{id}|/api/v1/users/{id}/orders|/api/v1/orders/{id}/items;onMiss.action=emit
```

`onMiss.value` defaults to empty and is set the same way (`onMiss.value=other`). The `path`, `onMiss.action`, and `onMiss.value` keys are read by the HTTP handler in `Init` and routed into `pathContext`, the same way the existing `exemplars` option is handled. They still pass through `ParseContextOptions` with the rest of the option list, but that function ignores names it does not recognize, so they add no context labels.

Kept for parity, but hard to read. The docs recommend the structured form below.

Structured form in the dynamic metrics ConfigMap introduced by CFP-30788. The Helm chart ships this as `cilium-dynamic-metrics-config` with data key `dynamic-metrics.yaml` (see `install/kubernetes/cilium/templates/cilium-dynamic-metrics-configmap.yaml`). Schema:

```yaml
metrics:
- name: http
  contextOptions:
    - name: sourceContext
      values: [pod]
    - name: destinationContext
      values: [pod]
  pathContext:
    templates:
      - /api/v1/users/{id}
      - /api/v1/users/{id}/orders
      - /api/v1/orders/{id}/items
    onMiss:
      action: emit          # emit | drop_sample
      value: ""             # label value when action == emit
```

`onMiss.action` says what happens when nothing matches:

* `emit` (default): emit the sample with `path` set to `onMiss.value`. Operator picks the value. Typical choices are `""` (empty bucket) or `"other"` (named bucket).
* `drop_sample`: do not emit the sample. Use this if only templated routes matter. `onMiss.value` is ignored with this action.

### Helm examples

The agent takes one config mechanism at a time: setting both static and dynamic metrics fails validation with "cannot configure both static and dynamic Hubble metrics" (`pkg/hubble/metrics/cell/cell.go`). Each example uses one.

Static string in `values.yaml`:

```yaml
hubble:
  metrics:
    enabled:
      - dns
      - "http:sourceContext=pod;destinationContext=pod;path=/api/v1/users/{id}|/api/v1/users/{id}/orders;onMiss.action=emit"
```

The same with `--set`. Helm's `--set` parser uses `{`, `}`, and `,` as list syntax and does not error on unescaped ones: it exits zero and silently truncates the value at the first unescaped `}` or `,`. Braces must be escaped, the same way the metrics docs already escape commas in `labelsContext` values:

```sh
--set 'hubble.metrics.enabled={dns,http:sourceContext=pod;destinationContext=pod;path=/api/v1/users/\{id\}|/api/v1/users/\{id\}/orders;onMiss.action=emit}'
```

`--set-json` needs no escaping and is what the docs will recommend for CLI installs:

```sh
--set-json 'hubble.metrics.enabled=["dns","http:sourceContext=pod;destinationContext=pod;path=/api/v1/users/{id}|/api/v1/users/{id}/orders;onMiss.action=emit"]'
```

Dynamic config map in `values.yaml`. The `hubble.metrics.dynamic.config.content` list renders under the `metrics:` key of `dynamic-metrics.yaml` (`install/kubernetes/cilium/templates/cilium-dynamic-metrics-configmap.yaml`):

```yaml
hubble:
  metrics:
    enabled: ~
    dynamic:
      enabled: true
      config:
        createConfigMap: true
        content:
          - name: http
            contextOptions:
              - name: sourceContext
                values: [pod]
              - name: destinationContext
                values: [pod]
            pathContext:
              templates:
                - /api/v1/users/{id}
                - /api/v1/users/{id}/orders
              onMiss:
                action: emit
                value: ""
```

There is no readable plain `--set` form for the structured content. Use a values file, `--set-json` on `hubble.metrics.dynamic.config.content`, or a self-managed ConfigMap (`createConfigMap: false` plus `configMapName`).

### Bounding cardinality

The matcher can only emit values from a fixed set: the N template strings, plus `onMiss.value` when `onMiss.action` is `emit`. Nothing derived from the request reaches the label, including on the error path, since an unparseable URL is a miss. A metric can therefore never expose more than `N + 1` distinct `path` values with `emit`, or `N` with `drop_sample`. The set is known at config load and identical across agent restarts.

That leaves N itself. Config validation caps the template list at 100 entries per metric. The cap is a constant, not a config knob. Raising it later is backwards compatible, lowering it is not, and real route lists sit well under it.

An earlier draft had a runtime `maxCardinality` cap that admitted label values first come, first served. It is gone: admission by arrival order means the same config can expose different label sets across agent restarts, and with template strings as label values there is nothing left for a runtime cap to catch.

### Self-observability

Two small metrics to verify the matcher is doing what the operator expects:

* `hubble_http_metric_path_template_match_total{result="hit|miss"}`
  Counts the outcome of each matcher run. A low hit ratio means the templates do not match real traffic.
* `hubble_http_metric_path_template_load_errors_total`
  Counts config loads rejected by `pathContext` validation.

### Backwards compatibility

With `pathContext` unset, metric shapes are identical to today. Dashboards, recording rules, and alerts are not affected.

With `pathContext` set, an extra `path` label appears. Queries that aggregate over the metric have to account for it.

`pathContext` is additive in the config schema. The dynamic config is parsed with the standard YAML decoder, so older Hubble agents reading a newer config map will ignore the unknown field rather than fail.

Toggling `pathContext` on or off changes the metric's label set. The dynamic config watcher already rejects reloads that change a registered metric's label set (`validateMetricConfig` in `pkg/hubble/metrics/metric_config_watcher.go`); this CFP extends that check to `pathContext` presence, so turning the feature on or off requires an agent restart, like changing context options does today. Editing the template list inside an active `pathContext` keeps the label names stable and reloads cleanly. On reload, series for removed templates are deleted from the registry, so the exposed values track the active config.

## Impacts / Key Questions

### Impact: Cardinality

The feature was rejected once before on cardinality grounds. The bound is structural: the `path` label takes at most `N + 1` distinct values per metric with `onMiss.action: emit`, or `N` with `drop_sample`, fixed at config load (see Bounding cardinality).

The number to plan capacity around is multiplicative, not additive. `path` multiplies the series that already exist for the metric: with `sourceContext=pod` and `destinationContext=pod`, every pod pair talking HTTP can produce up to `N + 1` path series where it produced one. Fewer templates mean fewer series; the template list is the operator's control for that.

A catch-all template like `/{anything...}` is not a cardinality problem. It matches every path and emits the single value `/{anything...}`: one bucket. The failure mode is a label that says nothing, not series growth.

### Key Question: Where do templates come from

#### Option 1: Operator-supplied template list

Default. Templates live in the metrics config.

Pros:

* Works whether or not L7 CNPs are in use.
* One list, easy to review in a PR.
* Easy to unit test.

Cons:

* If HTTP routes already live in L7 CNPs, that is a second list to maintain.

#### Option 2: Templates from L7 CiliumNetworkPolicy

Suggested on the issue by @apt-itude. If L7 HTTP rules already exist in `CiliumNetworkPolicy`, reuse those route patterns as the metric templates.

Pros:

* One source of truth in clusters that already use L7 CNPs.
* No extra config.

Cons:

* Editing a CNP would change metric labels. Not obvious to whoever is editing the policy.
* Does not help clusters that get L7 visibility some other way.
* CNP HTTP rules use regex and prefix matching. They do not map cleanly to a template that produces a stable label. The mapping has to be designed and will probably restrict which CNP rules are usable.

#### Recommendation

Ship Option 1. It is smaller and does not depend on L7 CNPs being in use. Option 2 can come later as a `templateSource: ciliumNetworkPolicy` flag once we know what the CNP-to-template mapping should look like.

### Key Question: What's the default when nothing matches

Default: `onMiss.action: emit` with `onMiss.value: ""`. The request is still counted. The empty bucket avoids adding a new named series unless the operator asks for one.

`drop_sample` is not the default, because silently dropping samples hides traffic the operator probably expects to see counted.

`onMiss.value: "other"` is supported. A non-empty value adds a series that competes with the templated ones. That choice belongs to the operator.

### Impact: Helm string config readability

The semicolon-and-pipe string format for `hubble-metrics` is already hard to read, and passing path templates through `helm --set` needs brace escaping on top (see Helm examples). Anyone turning on path labels should be on the dynamic config map from CFP-30788. The string form stays for parity, but the docs will point at YAML.
