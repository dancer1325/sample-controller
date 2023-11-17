# sample-controller

Example of how to build a kube-like controller with a single type.

## Details

Simple controller for watching 'Foo' resources
defined with a CustomResourceDefinition (CRD).

* Basic operations to perform
    * Register a new custom resource of type `Foo` -- via CustomResourceDefinition --.
    * Create/get/list instances of your new resource type `Foo`.
    * Set up a controller on resource handling create/update/delete events.

* Ways to generate a typed client, informers, listeners and deep-copy functions
    * Use the generators in [k8s.io/code-generator](https://github.com/kubernetes/code-generator)
    * Use the `./hack/update-codegen.sh` script
        * generate the following files & directories:
            * `pkg/apis/samplecontroller/v1alpha1/zz_generated.deepcopy.go`
            * `pkg/generated/`
        * If you want to create your own controller based on this implementation ->
            * not copy these files
            * generate your own `./hack/update-codegen.sh`

* [Here](docs/controller-client-go.md) , you can find the details
  * [client-go library](https://github.com/kubernetes/client-go/tree/master/tools/cache) extensively used
  

## How to fetch sample-controller and its dependencies

Ways to fetch this demo and its dependencies

### Via godep and `$GOPATH` -- without using go 1.11 modules -- 

* [godep](https://github.com/tools/godep)
* Used during years by own Kubernetes

```sh
go get -d k8s.io/sample-controller
cd $GOPATH/src/k8s.io/sample-controller
godep restore
```

### Via go 1.11 modules

* === `GO111MODULE=on`

```sh
git clone https://github.com/kubernetes/sample-controller.git
cd sample-controller
```

* If you intend to generate code -> you will also need the code-generator repo to exist
  * === create and populate the `vendor` directory
  * `go mod vendor`

## How to run

* Prerequisite
  * Kubernetes cluster version v1.9+
    * Reason: Sample-controller uses `apps/v1` deployments

```sh
# assumes you have a working kubeconfig, not required if operating in-cluster
go build -o sample-controller .
./sample-controller -kubeconfig=$HOME/.kube/config

# create a CustomResourceDefinition
kubectl create -f artifacts/examples/crd-status-subresource.yaml

# create a custom resource of type Foo
kubectl create -f artifacts/examples/example-foo.yaml

# check deployments created through the custom resource
kubectl get deployments
```


## How to clean up

You can clean up the created CustomResourceDefinition with:
```sh
kubectl delete crd foos.samplecontroller.k8s.io
```

## Notes
* go-get or vendor this package as `k8s.io/sample-controller`-- TODO: ? --
* Refer in [kubernetes/kubernetes](kubernetes/staging/src/k8s.io/sample-controller)
* The [group](https://kubernetes.io/docs/reference/using-api/#api-groups) version of the custom resource in `crd.yaml` is `v1alpha`
  * Using [CRD Versioning](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/) ->  it can be evolved to a stable API version, `v1` 


### Define types

Each instance of your custom resource -> has an attached Spec
* == arbitrary key-value data which
    * specifies the configuration/behavior
    * provide data format validation
        * use the [`CustomResourceValidation`](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#validation) feature
        * in the form of a [structured schema](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema) is mandatory to be provided for `apiextensions.k8s.io/v1`
        * Example: `spec.replicas` in [`crd.yaml`](./artifacts/examples/crd.yaml)
* defined via a `struct{}`
    * Example: custom resource for a Database
        * `json:` is a tag
            * required on all user facing fields within your type
            * API types contain normally only user facing fields
            * If it's omitted from the field -> Kubernetes generators consider the field
                * as internal
                * not exposed in their generated external output === not included in a generated CRD schema

``` go
type DatabaseSpec struct {
	Databases []string `json:"databases"`
	Users     []User   `json:"users"`
	Version   string   `json:"version"`
}

type User struct {
	Name     string `json:"name"`
	Password string `json:"password"`
}
```

### Subresources

* [subresources](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#subresources)
* support `/status` and `/scale`
* Example: CRD in [`crd-status-subresource.yaml`](./artifacts/examples/crd-status-subresource.yaml)
    * -> [`UpdateStatus`](./controller.go) can be used by the controller to update only the status part of the custom resource
        * Reason: [Kubernetes API conventions](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status) TODO: ?