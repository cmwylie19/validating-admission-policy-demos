# Validating Admission Policy

- [Create Cluster](#create-cluster)
- [Demo](#demo)
- [Background](#background)
- [links](#links)

## Create Cluster

We must enable the `ValidatingAdmissionPolicy` feature gate. We will use Kind for this demo.

```bash
cat  >config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  "ValidatingAdmissionPolicy": true
runtimeConfig:
  "admissionregistration.k8s.io/v1alpha1": true
nodes:
- role: control-plane
  image: kindest/node:v1.26.0@sha256:691e24bd2417609db7e589e1a479b902d2e209892a10ce375fab60a8407c7352
EOF

kind create cluster --config config.yaml
```


## Demo 

Create a `ValidatingAdmissionPolicy` to make sure that all pods have a label `pepr.dev/netpol` set to `enabled`


```yaml
kubectl create -f -<<EOF
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "pepr-pod-labels"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  validations:
  - expression: "object.metadata.labels['pepr.dev/netpol'] =='enabled'"
    message: "'only pepr.dev/netpol == enabled labeled pods are allowed. '"
---
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "pepr-pod-labels-binding"
spec:
  policyName: "pepr-pod-labels"
  matchResources: 
    namespaceSelector: {} # all namespaces
EOF
```

Test the `ValidatingAdmissionPolicy` by creating a pod without the label.

```bash
k run no-label --image=nginx --restart=Never

The pods "no-label" is invalid: : ValidatingAdmissionPolicy 'pepr-pod-labels' with binding 'pepr-pod-labels-binding' denied request: expression 'object.metadata.labels['pepr.dev/netpol'] =='enabled'' resulted in error: no such key: pepr.dev/netpol
```

Create a pod with the the wrong value for the label.

```bash
k run wrong-label --image=nginx -l pepr.dev/netpol=disabled

The pods "wrong-label" is invalid: : ValidatingAdmissionPolicy 'pepr-pod-labels' with binding 'pepr-pod-labels-binding' denied request: 'only pepr.dev/netpol == enabled labeled pods are allowed. 
``` 

Create the correct pod with the label.

```bash
k run po --image=nginx -l pepr.dev/netpol=enabled

pod/po created
```

Clean Up 

```bash
kind delete clusters --all
```


## Background

_You can skip straight to the [demo](#demo) if you don't care about the background._

A policy is made up of 3 resources:
- The `ValidatingAdmissionPolicy` describes the abstract logic of a policy (think: "this policy makes sure a particular label is set to a particular value").

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: less-than-five
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas <= 5"
```

- A `ValidatingAdmissionPolicyBinding` links the above resources together and provides scoping. If you only want to require an owner label to be set for Pods, the binding is where you would specify this restriction.

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: less-than-five-binding
spec:
  policyName: less-than-five
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: test
```

- [NOT REQUIRED] A parameter resource provides information to a ValidatingAdmissionPolicy to make it a concrete statement (think "the owner label must be set to something that ends in .company.com"). A native type such as ConfigMap or a CRD defines the schema of a parameter resource. `ValidatingAdmissionPolicy` objects specify what Kind they are expecting for their parameter resource.


```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingAdmissionPolicy
metadata:
  name: "replicalimit-policy.example.com"
spec:
  failurePolicy: Fail
  paramKind:
    apiVersion: rules.example.com/v1
    kind: ReplicaLimit
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas <= params.maxReplicas"
      reason: Invalid
---
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "replicalimit-binding-test.example.com"
spec:
  policyName: "replicalimit-policy.example.com"
  validationActions: [Deny]
  paramRef:
    name: "replica-limit-test.example.com"
    namespace: "default"
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: test
---
apiVersion: rules.example.com/v1
kind: ReplicaLimit
metadata:
  name: "replica-limit-test.example.com"
  namesapce: "default"
maxReplicas: 3
```


## Links

- [validating admission policies](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)


[TOP](#validating-admission-policy)
