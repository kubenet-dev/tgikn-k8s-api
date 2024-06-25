# tgikn kubernetes api

background information on the exercise

[kubebuilder](https://book.kubebuilder.io/introduction)
[crds](https://book.kubebuilder.io/reference/generating-crd)

## getting started

### option 1

start the devcontainer

https://codespaces.new/kubenet-dev/tgikn-k8s-api

### option 2

install pre-requiesites

install go

[go](https://go.dev/doc/install)

create a cluster in order to test the apis we create

[kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

install kubectl

[kubectl](https://kubernetes.io/docs/tasks/tools/)

## setup your project

make a directory of the project with apis directory

```
mkdir test; cd test
```

```
mkdir -p apis/
```
initialize the go project

```
go mod init github.com/dummy/test
```

create a Makefile with the following content

```bash
touch Makefile
```

```bash
## Location to install dependencies to
LOCALBIN ?= $(shell pwd)/bin
$(LOCALBIN):
	mkdir -p $(LOCALBIN)

## Tool Binaries
CONTROLLER_GEN ?= $(LOCALBIN)/controller-gen
CONTROLLER_TOOLS_VERSION ?= v0.15.0

# Setting SHELL to bash allows bash commands to be executed by recipes.
# Options are set to exit when a recipe line exits non-zero or a piped command fails.
SHELL = /usr/bin/env bash -o pipefail
.SHELLFLAGS = -ec

.PHONY: manifests

all: manifests

.PHONY: manifests
manifests: controller-gen ## Generate WebhookConfiguration, ClusterRole and CustomResourceDefinition objects.
	mkdir -p artifacts
	$(CONTROLLER_GEN) rbac:roleName=manager-role crd paths="./apis/..." output:crd:artifacts:config=artifacts

.PHONY: controller-gen
controller-gen: $(CONTROLLER_GEN) ## Download controller-gen locally if necessary.
$(CONTROLLER_GEN): $(LOCALBIN)
	echo $(LOCALBIN)
	test -s $(LOCALBIN)/controller-gen || GOBIN=$(LOCALBIN) go install sigs.k8s.io/controller-tools/cmd/controller-gen@$(CONTROLLER_TOOLS_VERSION)
```

```bash
make manifests
```

## create your first api

group: a collection of related functionality
version: each group has one or more versions, which, as the name suggests, allow us to change how an API works over time
kind: what your api is about
resources: (plural/lowercase of the kind) how you address the resource from http and storage (PUT, POST, PATCH, etc)
subresource: tests/status

[gvk](https://book.kubebuilder.io/cronjob-tutorial/gvks)

create a directory where we store your api artifacts

```
mkdir -p apis/foo/v1alpha1
```

create a doc.go file and the test_types.go file containing the api

```
touch apis/foo/v1alpha1/doc.go
touch apis/foo/v1alpha1/test_types.go
```

```go
// +kubebuilder:object:generate=true
// +groupName=foo.example.com
// Package v1alpha1 is the v1alpha1 version of the API.
package v1alpha1
```

```go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// TestSpec defines the desired state of the Test resource
type TestSpec struct {
}

// TestStatus defines the observed state of the Test resource
type TestStatus struct {
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// Test is the Schema for the Test API
type Test struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   TestSpec   `json:"spec,omitempty"`
	Status TestStatus `json:"status,omitempty"`
}

// TestList contains a list of Tests
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type TestList struct {
	metav1.TypeMeta `json:",inline" yaml:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Test `json:"items"`
}
```

run go mod tidy to resolve the dependencies

```
go mod tidy
```

generate your api

```
make manifests
```

```
k apply -f artifacts
```

```yaml
kubectl apply -f - <<EOF
apiVersion: foo.example.com/v1alpha1
kind: Test
metadata:
  name: my-first-test
EOF
```

## adding fields with validation

[crd validation](https://book.kubebuilder.io/reference/markers/crd-validation)

```go
type TestSpec struct {
    // +kubebuilder:validation:MaxLength=15
    // +kubebuilder:validation:MinLength=3
	// FieldA defines the name of field A
    FieldA string `json:"fieldA,omitempty"`
}
```

make manifests and apply to the cluster

```
make manifests
k apply -f artifacts
```

create an object with an empty fieldA in spec -> acceppted

```yaml
kubectl apply -f - <<EOF
apiVersion: foo.example.com/v1alpha1
kind: Test
metadata:
  name: my-first-test
spec:
EOF
```

create an object with a fieldA but with 1 char in spec -> rejected

```yaml
kubectl apply -f - <<EOF
apiVersion: foo.example.com/v1alpha1
kind: Test
metadata:
  name: my-first-test
spec:
  fieldA: w
EOF
```

create an object with a fieldA but with 3 char in spec -> accepted

```yaml
kubectl apply -f - <<EOF
apiVersion: foo.example.com/v1alpha1
kind: Test
metadata:
  name: my-first-test
spec:
  fieldA: wim
EOF
```

## decorations printcolumn, categories

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="FIELDA",type="string",JSONPath=".spec.fieldA"
// +kubebuilder:resource:categories={knet}
// Test is the Schema for the Test API
type Test struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   TestSpec   `json:"spec,omitempty"`
	Status TestStatus `json:"status,omitempty"`
}
```

```
make manifests
k apply -f artifacts/foo.example.com_tests.yaml
```


## add additional constraints

[crd validation](https://book.kubebuilder.io/reference/markers/crd-validation)

```go
type TestSpec struct {
    // +kubebuilder:validation:MaxItems=2
	// ListA defines a list of A
    ListA []string `json:"listA,omitempty"`

	// +kubebuilder:validation:Enum=unknown;gnmi;netconf;noop;ssh;
	// +kubebuilder:default:="gnmi"
	// Protocol defines the protocol to connect to the device
	Protocol Protocol `json:"protocol"`

	// +kubebuilder:validation:Pattern=`(([0-9]|[1-9][0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-5])|((:|[0-9a-fA-F]{0,4}):)([0-9a-fA-F]{0,4}:){0,5}((([0-9a-fA-F]{0,4}:)?(:|[0-9a-fA-F]{0,4}))|(((25[0-5]|2[0-4][0-9]|[01]?[0-9]?[0-9])\.){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9]?[0-9])))`
	// Address defines the ip address to connect to the host
	Address string `json:"address"`

	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=200
	// +kubebuilder:default:=100
	Priority   *uint8 `json:"priority,omitempty"`
}
```

