# Direct Resource API Validation


# TL;DR

Config Connector Direct Resource shall have each `spec` field validated.


# Basic Rules

Each Config Connector resource `spec` field shall be validated by at least one of the following approach:


## 1. CRD validation

* This throws standard OpenAPI validation errors without requiring the Config Connector controller to run.
* This validation shall take the responsibility to give users a cheap and self-guiding check. It does not contain complicated logic. 


## 2. Config Connector controller validation


* This throws the Config Connector-defined errors in runtime.
* This validation shall take the responsibility to guardrail config issues that otherwise could cause ambiguous or unexpected Config Connector behavior.
* We shall build and publish a Config Connector error code table to future explain the issue and give a triage guide.


## 3. GCP Service Validation


* This throws GCP defined errors in runtime. Config Connector shall propagate the GCP response errors to the Config Connector object’s `status` field. 
* This validation shall take the responsibility to guide users on fixing the issues due to GCP service requirements, like `SpannerInstance` has `spec.processingUnit` whose value can only increase but not decrease. 


# CRD validation


## Rule 1: Required/Optional

Required field should use tag `+required`. 

Optional field does not need the tag.


```
type CloudBuildWorkerPoolSpec struct {
  // +required   
  PrivatePoolConfig *PrivatePoolV1Config `json:"privatePoolV1Config,omitempty"`
}
```


## Rule 2: Immutable field

Immutable field should be validated via CEL rule using kubebuilder tag 


```
type PrivatePoolV1Config_NetworkConfigSpec struct {
        // +kubebuilder:validation:XValidation:rule="self == oldSelf",message="the field is immutable"
        // Immutable. The network definition that the workers are peered
        //  to. If this section is left empty, the workers will be peered to
        //  `WorkerPool.project_id` on the service producer network.
        PeeredNetworkRef refv1beta1.ComputeNetworkRef `json:"peeredNetworkRef,omitempty"`
}
```

Note: 


* Kubernetes supports CEL since 1.26 which is the oldest GKE cluster version (written at ). Using CEL shall not cause GKE cluster version issues, but we shall reevaluate this when the condition changes.
* Some Config Connector resources use webhook to validate immutable fields. We can continue using that, but for external contributors CEL is much easier for self-learning. 

We only apply CEL on immutable fields.** More complicated CEL validation requires further discussion.**

One future improvement can be having generate-types detect the “Immutable.” comment and add the kubebuilder tag automatically. 


## Rule 3: Parent

Parent field should be required and immutable.


### Required

Suggest using the following `inline` struct


```
type CloudBuildWorkerPoolSpec struct {
  // +required   
  Parent `json:",inline"`
}

type Parent struct {
    // +required      
    ProjectRef *refv1beta1.ProjectRef `json:"projectRef"`

    // +required
    Location string `json:"location"`
}
```


### Immutable

Since most parent is a reference to another resource which can either be `<kind>Ref.external `or `<kind>Ref.name` , the immutable validation needs to be done at the controller validation level. For example, switching `<kind>Ref.external `and `<kind>Ref.name `are allowed if referring to the same resource.

See this [resource reference guide](./resource-reference.md) (todo update once PR merged) for general rules;

See this [external reference guide](./external-reference.md#statusexternalref-format) for parent validation with `status.externalRef`.   


## Rule 4: No `anyOf`, `oneOf`, or` allOf`

Short answer: Not now.

Full answer: 

OpenAPI supports `oneOf`, `anyOf` and `allOf`. Thus, in theory these should be the CRD level rules. The Direct Resource CRD is auto-generated by controller-gen, which only reads the kubebuilder tag rules. But the kubebuilder tags [do not (and most likely won’t)](https://github.com/kubernetes-sigs/controller-tools/issues/461) support `oneOf`, `anyOf` or `allOf`.

To set this as CRD level rule, Config Connector shall build its own CRD transformer.  Ideally, this could be a Config Connector tag similar to kubebuilder, but this requires integrating with controller-gen which is a non-trivial amount of work. Other options like webhook, OpenAPI schema modifier are not self-explaining and easy enough to use, considering that Direct Resource shall be open to external contributors. Thus, Config Connector does not use CRD level validation for `oneOf`, `anyOf` and `allOf`.

One future improvement can be having a Direct Resource tag that adds CRD level checks for   `oneOf`, `anyOf` or `allOf`.


```
// +Config Connector:validation:oneOf:serviceAttachmentRef,targetGRPCProxyRef,targetHTTPProxyRef,targetHTTPSProxyRef"
type target struct {
        ServiceAttachmentRef *refs.ComputeServiceAttachmentRef `json:"serviceAttachmentRef,omitempty"`

        TargetGRPCProxyRef *refs.ComputeTargetGrpcProxyRef `json:"targetGRPCProxyRef,omitempty"`

        TargetHTTPProxyRef *refs.ComputeTargetHTTPProxyRef `json:"targetHTTPProxyRef,omitempty"`

        TargetHTTPSProxyRef *refs.ComputeTargetHTTPSProxyRef `json:"targetHTTPSProxyRef,omitempty"`
}
```



# Controller validation

Advanced Kubernetes contributors can use webhook to validate the resource CRs. 

To easily ramp-up external contributors, we recommend using the controller-level validation instead. This validation mostly happens on Adapter [initialization](https://github.com/GoogleCloudPlatform/k8s-config-connector/blob/91aac5186eb0aa2c5c3def94d7a7c79f948ac9a9/pkg/controller/direct/directbase/directbase_controller.go#L226), GCP object [creation](https://github.com/GoogleCloudPlatform/k8s-config-connector/blob/91aac5186eb0aa2c5c3def94d7a7c79f948ac9a9/pkg/controller/direct/directbase/directbase_controller.go#L283) and [update](https://github.com/GoogleCloudPlatform/k8s-config-connector/blob/91aac5186eb0aa2c5c3def94d7a7c79f948ac9a9/pkg/controller/direct/directbase/directbase_controller.go#L291) steps. 


## Rule 1: Resource Reference

Config Connector shall validate the resource reference based on the [resource reference guide](./resource-reference.md). 


## Rule 2: Resource ID

`spec.resourceID` is an optional and  immutable field. 

Config Connector shall validate the `resourceID` (if present) matches the `status.externalRef` (if present) based on the  reconciliation 4 steps.
