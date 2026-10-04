# CFP-XXXXX: Policy Selector Expressions

**SIG: SIG-Policy**

**Begin Design Discussion:** 2026-05-04

**Cilium Release:** TBD

**Authors:** Deepesh Pathak

**Status:** Draft

## Summary

This CFP proposes extending [Cilium's policy selectors](https://github.com/cilium/cilium/blob/f7c0ac66d9d781ecb5bd72c59025620b01c912d8/pkg/policy/api/selector.go#L17), allowing users to write complex selection expressions than what is supported today with [`LabelSelector`](https://github.com/cilium/cilium/blob/f7c0ac66d9d781ecb5bd72c59025620b01c912d8/pkg/k8s/slim/k8s/apis/meta/v1/types.go#L267) - specifically operators such as prefix, suffix, and substring matching, along with full boolean composition.

## Motivation

Currently cilium's policy selector type([`EndpointSelector`](https://github.com/cilium/cilium/blob/f7c0ac66d9d781ecb5bd72c59025620b01c912d8/pkg/policy/api/selector.go#L17)) wraps the Kubernetes [`LabelSelector`](https://github.com/cilium/cilium/blob/f7c0ac66d9d781ecb5bd72c59025620b01c912d8/pkg/k8s/slim/k8s/apis/meta/v1/types.go#L267) type, which supports only equality-based matching (`matchLabels`) and a small fixed set of set-membership operators (`In`, `NotIn`, `Exists`, `DoesNotExist`). For most workloads this is sufficient, but there are common cases where users need more expressive matching.

While some of these use-cases can be hacked with the current selector semantics(eg. splitting selection constraints across multiple selectors, enumerating every concrete value in `matchExpression` entry etc.), they become very verbose, brittle and hard to maintain. Moreover, defining the selection constraints in this way sometimes comes with runtime cost in cilium-agent(eg. high selector cardinality), that might not be apparent to the user.

### User stories

- [Support multiple endpoint selectors in CNP rules](https://github.com/cilium/cilium/issues/45682)

## Goals

- Allow users to write richer label matching conditions in policy selectors that go beyond equality and set membership.
- Allow users to compose multiple boolean expressions in a selector.
- Preserve full backward compatibility: existing policies that use only `matchLabels` / `matchExpressions` should continue to work without change.

## Non-Goals

- Migrate/Replace existing policy label selector semantics.

## Proposal

### Overview

The proposal is to add a new optional field in policy selector type [`EndpointSelector`](https://github.com/cilium/cilium/blob/f7c0ac66d9d781ecb5bd72c59025620b01c912d8/pkg/policy/api/selector.go#L17) that allows users to write a serialized structured match expression. The expression is validated and compiled during policy import and used as a requirement in policy [`LabelSelector`](https://github.com/cilium/cilium/blob/f7c0ac66d9d781ecb5bd72c59025620b01c912d8/pkg/k8s/slim/k8s/apis/meta/v1/types.go#L267) type for matching labels.

```go
// EndpointSelector is a wrapper for k8s LabelSelector.
type EndpointSelector struct {
	*slim_metav1.LabelSelector `json:",inline"`

	// MatchCELExpression is an optional, serialized, boolean CEL expression
	// that provides additional label match conditions for this selector.
	// When set, an endpoint must satisfy this expression **AND** the
	// k8s LabelSelector constraints.
	//
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:MaxLength=4096
	MatchCELExpression *string `json:"matchCELExpression,omitzero"`
}
```

## Impacts / Key Questions

### Expression Language

The central design aspect for the implementation is to decide the expression language to use.

#### Option 1: Custom expression language

The idea with this option is to define a custom selection expression language. As a reference, this is how [Calico selector expressions](https://docs.tigera.io/calico-enterprise/latest/reference/resources/networkpolicy#selector) are implemented. This will involve designing the SPEC for the language and writing the compiler.

##### Pros

- Match evaluation can be made zero-allocation with a very low overhead(see POC benchmarks).
- The operator set and language spec will be narrow, implementing exactly the requirements for label selectors which limits the surface area for misuse and bugs.
- No new external dependencies.

##### Cons

- The parser implementation needs to be maintained in cilium source tree.
- Language spec is not standard and users must learn a new format.

#### Option 2: CEL Expression

A more natural option for the use-case is to use CEL(Common Expression Language) based expression for label selectors. Cilium already imports the `cel-go` library and has experimental support for CEL based expression language in hubble filters. This option will give us the opportunity to build a strong foundation for CEL usage in Cilium project, allowing easier adoption for similar usecases in future.

##### Pros

- CEL is a CNCF standard, heavily utilized in Kubernetes ecosystem (admission validation, DRA device selectors etc.).
- Users already using CEL in other contexts will find the syntax familiar.
- CEL's type system, inbuilt macro's, operators and extension model allow for a rich selector expression language.
- CEL has a well-defined cost model and supports configurable evaluation cost limits, which can be used to bound the runtime cost of an expression and prevent accidental or malicious policy expressions.

##### Cons

- Parse and match cost is significantly higher, making policy computation/identity updates more expensive.

##### Implementation (CEL Expression)

The draft implementation of CEL based selection expression in policy selectors is present here: https://github.com/cilium/cilium/pull/46199

The environment for LabelSelector CEL expression provides a `label(<source>:<key>)` macro
to lookup value of a key(with/without source prefix) against the Labels being evaluated for selection.
The return type of this macro is the label value(type: `string`) wrapped in [CEL's optional type](https://pkg.go.dev/github.com/google/cel-go/cel#OptionalTypes)

> Currently the [internal representation of identity labels](https://github.com/fristonio/cilium/blob/4b557bd47740671e51bd5aa09cf6677eae4e6b94/pkg/policy/selectorcache.go#L31) is implemented as label array instead of a map.
> Exposing a high level macro for selection expression provides us the opportunity to simplify internal types
> without any user impact in the future(out of scope for this proposal).

Example Usage:

| Description                                | Sample Expression                                                                     |
| ------------------------------------------ | ------------------------------------------------------------------------------------- |
| Check if the key exists in identity labels | `label("k8s:app").hasValue()`                                                         |
|                                            | `label("any:missing") != optional.none()`                                             |
| Value of a key from identity labels        | `label("k8s:app").value()`                                                            |
|                                            | `label("k8s:app").orValue("test")`                                                    |
|                                            | `label("k8s:app") == optional.of("myapp")`                                            |
| Set membership/comparision operations      | `label("k8s:env").value() in ["prod", "staging", "dev"]`                              |
|                                            | `[label("k8s:env").value(), label("app").value()] == ["prod", "myapp"]`               |
| Label value string operations              | `label("k8s:env").orValue("").startsWith("prod")`                                     |
|                                            | `label("k8s:env").orValue("").endsWith("-us-east")`                                   |
|                                            | `label("k8s:env").orValue("").contains("db-staging")`                                 |
| Boolean composition                        | `label("k8s:env").value() == "dev" \|\| label("k8s:env").value() == "staging"`        |
|                                            | `label("k8s:env") == optional.of("prod") && label("k8s:app") == optional.of("myapp")` |
|                                            | `label("k8s:env") != optional.of("prod")`                                             |
| Iterators                                  | `["k8s:app", "k8s:env"].all(k, label(k).hasValue())`                                  |
|                                            | `["k8s:app", "k8s:env"].map(k, label(k).orValue("")) == ["myapp", "prod"]`            |

### Performance Implication (POC)

A rough POC implementation for both options is [present here](https://github.com/fristonio/cilium/commits/pr/fristonio/poc/policy-selector-expressions/)

#### Parse/Compile

|                   | ns/op   | B/op    | allocs/op |
| ----------------- | ------- | ------- | --------- |
| Custom Expression | 3,585   | 7,704   | 72        |
| CEL Expression    | 182,470 | 184,860 | 2,716     |

#### Match

|                   | ns/op | B/op | allocs/op |
| ----------------- | ----- | ---- | --------- |
| Custom Expression | 17.6  | 0    | 0         |
| CEL Expression    | 320   | 512  | 7         |

As mentioned before, CEL based selector expression have a very high cost as compared to a custom implementation. However, this is a reasonable trade-off, given for most usecases and simple expressions user can continue to use `LabelSelector` type.

### References

- [Calico policy selector parser](https://github.com/projectcalico/calico/tree/master/libcalico-go/lib/selector)
- [CEL Golang library extensions](https://github.com/google/cel-go/tree/master/ext)
- [Kubernetes CEL API](https://kubernetes.io/docs/reference/using-api/cel/)
- [Kubernetes DRA CEL environment](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/dynamic-resource-allocation/cel)
