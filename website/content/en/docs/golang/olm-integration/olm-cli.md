---
title: Running your Operator with OLM
linkTitle: Running with OLM
weight: 20
---

The [Operator Lifecycle Manager (OLM)][olm] is a set of cluster resources that
manage the lifecycle of an Operator. The Operator SDK provides an entrypoint for
deploying and deleting your Operator using an OLM-enabled Kubernetes cluster
through `operator-sdk run --olm`, added in [v0.15.0][sdk-release-v0.15.0]

This document assumes you are familiar with OLM and related terminology, and have
read the SDK-OLM integration [design proposal][sdk-olm-design].

**Note:** before continuing, please read the [caveats](#caveats) section below.

## Setup

The SDK assumes OLM is already installed and running on your cluster. If not,
you can install OLM by running [`operator-sdk olm install`][cli-olm-install].
You can check the status of OLM by running [`operator-sdk olm status`][cli-olm-status].

The SDK also assumes you have a valid [Operator bundle][operator-bundle] available
on disk. If not, you can create one using [`operator-sdk generate csv --update-crds`][cli-generate-csv],
which will create a [`ClusterServiceVersion` (CSV)][csv] in a versioned bundle
directory and a [package manifest][pkg-manifest] defining your Operator package,
and copy your `CustomResourceDefinition`'s to the new bundle directory.


## `operator-sdk run --olm` command overview

Let's look at the anatomy of the command:

```console
$ operator-sdk run --help
Run an Operator in a variety of environments

Usage:
  operator-sdk run [flags]

Flags:
      --kubeconfig string           The file path to kubernetes configuration file. Defaults to location specified by $KUBECONFIG, or to default file rules if not set
      --olm                         The operator to be run will be managed by OLM in a cluster. Cannot be set with another run-type flag
      --olm-namespace string        [olm only] The namespace where OLM is installed (default "olm")
      --operator-namespace string   [olm only] The namespace where operator resources are created in --olm mode. It must already exist in the cluster or be defined in a manifest passed to IncludePaths.
      --manifests string            [olm only] Directory containing package manifest and operator bundles.
      --operator-version string     [olm only] Version of operator to deploy
      --install-mode string         [olm only] InstallMode to create OperatorGroup with. Format: InstallModeType=[ns1,ns2[, ...]]
      --include strings             [olm only] Path to Kubernetes resource manifests, ex. Role, Subscription. These supplement or override defaults generated by run/cleanup
      --timeout duration            [olm only] Time to wait for the command to complete before failing (default 2m0s)
      ...
```

These flags correspond to options in the `OLMCmd` configuration model (names are similar if not identical):

```go
type OLMCmd struct {
  // KubeconfigPath is the local path to a kubeconfig. This uses well-defined
  // default loading rules to load the config if empty.
  KubeconfigPath string
  // OLMNamespace is the namespace in which OLM is installed.
  OLMNamespace string
  // OperatorNamespace is the cluster namespace in which operator resources
  // are created.
  // OperatorNamespace must already exist in the cluster or be defined in
  // a manifest passed to IncludePaths.
  OperatorNamespace string
	// ManifestsDir is a directory containing a package manifest and N bundles
	// of the operator's CSV and CRD's. OperatorVersion can be set to the
	// version of the desired operator version's subdir and Run()/Cleanup() will
	// deploy the operator version in that subdir.
	ManifestsDir string
	// OperatorVersion is the version of the operator to deploy. It must be
	// a semantic version, ex. 0.1.0.
	OperatorVersion string
	// InstallMode specifies which supported installMode should be used to
	// create an OperatorGroup. The format for this field is as follows:
	//
	// "InstallModeType=[ns1,ns2[, ...]]"
	//
	// The InstallModeType string passed must be marked as "supported" in the
	// CSV being installed. The namespaces passed must exist or be created by
	// passing a Namespace manifest to IncludePaths. An empty set of namespaces
	// can be used for AllNamespaces.
	// The default mode is OwnNamespace, which uses OperatorNamespace or the
	// kubeconfig default.
	InstallMode string
  // IncludePaths are path to manifests of Kubernetes resources that either
	// supplement or override defaults generated by methods of OLMCmd. These
	// manifests can be but are not limited to: RBAC, Subscriptions,
	// CatalogSources, OperatorGroups.
	//
	// Kinds that are overridden if supplied:
	// - CatalogSource
	// - Subscription
	// - OperatorGroup
	IncludePaths []string
	// Timeout dictates how long to wait for a REST call to complete. A call
	// exceeding Timeout will generate an error.
	Timeout time.Duration
}
```

Most of the above configuration options do not need explanation beyond their descriptions.
The following options require more clarification:

- `--operator-version`, on top of being a semantic version string corresponding to a
CSV version, should point to the directory containing your Operator bundle.
This path is `deploy/olm-catalog/<operator-name>/<operator-version>` in an SDK Operator project.
- `--include` can be used if you have an existing set of manifests outside your
bundle (ex. catalog manifests like a `Subscription`, `CatalogSource`, and/or `OperatorGroup`)
you wish to facilitate Operator deployment with.
  - Currently any manifests supplied to this command will be created with the same
  behavior of `kubectl create -f <manifest-path>`.
  - If a `Subscription` or `CatalogSource` are supplied, the other must be supplied
  since they are linked by field references.
- `--install-mode` configures the `spec.targetNamespaces` field of an `OperatorGroup`,
and supports the following modes (assuming your CSV does as well):
  - `OwnNamespace`: the Operator will watch its own namespace. This is the default.
  - `SingleNamespace`: the Operator will watch a namespace, not necessarily its own.
  - `MultiNamespace`: the Operator will watch multiple namespaces.
  Example: `--install-mode=MultiNamespace=my-ns-1,my-ns-2` will watch resources in namespaces `my-ns-1` and `my-ns-2`.
  - `AllNamespaces`: the Operator will watch all namespaces (cluster-scoped Operators).
  Example: `--install-mode=AllNamespaces=""` will watch resources in all namespaces given

## Examples

### Setup

Lets walk through creating a `memcached-operator` and enabling OLM on your cluster.

Follow the user guides for [Go][sdk-user-guide-go], [Ansible][sdk-user-guide-ansible],
or [Helm][sdk-user-guide-helm], depending on which Operator type you are interested in.

Now that we have a working, simple Operator, we can generate a new `v0.1.0` bundle:

**Note:** `operator-sdk generate csv` only officially supports Go Operators,
although it will generate a barebones CSV for Ansible and Helm Operators that _will_ require manual modification.

```console
$ operator-sdk generate csv --csv-version 0.1.0 --update-crds
INFO[0000] Generating CSV manifest version 0.1.0
INFO[0004] Required csv fields not filled in file deploy/olm-catalog/memcached-operator/0.1.0/memcached-operator.v0.1.0.clusterserviceversion.yaml:
	spec.keywords
	spec.provider
INFO[0004] Created deploy/olm-catalog/memcached-operator/0.1.0/memcached-operator.v0.1.0.clusterserviceversion.yaml
```

A bundle directory containing a CSV and all CRDs in `deploy/crds` and package manifest
have been created at `deploy/olm-catalog/memcached-operator/0.1.0` and
`deploy/olm-catalog/memcached-operator/memcached-operator.package.yaml`, respectively.

Ensure OLM is enabled on your cluster:

```console
# If OLM is not already installed, go ahead and install the latest version.
$ operator-sdk olm install
INFO[0000] Fetching CRDs for version "latest"           
INFO[0001] Fetching resources for version "latest"      
INFO[0007] Creating CRDs and resources                  
INFO[0007]   Creating CustomResourceDefinition "clusterserviceversions.operators.coreos.com"
INFO[0007]   Creating CustomResourceDefinition "installplans.operators.coreos.com"
INFO[0007]   Creating CustomResourceDefinition "subscriptions.operators.coreos.com"
INFO[0007]   Creating CustomResourceDefinition "catalogsources.operators.coreos.com"
INFO[0007]   Creating CustomResourceDefinition "operatorgroups.operators.coreos.com"
INFO[0007]   Creating Namespace "olm"                   
INFO[0007]   Creating Namespace "operators"             
INFO[0007]   Creating ServiceAccount "olm/olm-operator-serviceaccount"
INFO[0007]   Creating ClusterRole "system:controller:operator-lifecycle-manager"
INFO[0007]   Creating ClusterRoleBinding "olm-operator-binding-olm"
INFO[0007]   Creating Deployment "olm/olm-operator"     
INFO[0007]   Creating Deployment "olm/catalog-operator"
INFO[0007]   Creating ClusterRole "aggregate-olm-edit"  
INFO[0007]   Creating ClusterRole "aggregate-olm-view"  
INFO[0007]   Creating OperatorGroup "operators/global-operators"
INFO[0011]   Creating OperatorGroup "olm/olm-operators"
INFO[0011]   Creating ClusterServiceVersion "olm/packageserver"
INFO[0011]   Creating CatalogSource "olm/operatorhubio-catalog"
INFO[0011] Waiting for deployment/olm-operator rollout to complete
INFO[0011]   Waiting for Deployment "olm/olm-operator" to rollout: 0 of 1 updated replicas are available
INFO[0013]   Deployment "olm/olm-operator" successfully rolled out
INFO[0013] Waiting for deployment/catalog-operator rollout to complete
INFO[0013]   Waiting for Deployment "olm/catalog-operator" to rollout: 0 of 1 updated replicas are available
INFO[0018]   Deployment "olm/catalog-operator" successfully rolled out
INFO[0018] Waiting for deployment/packageserver rollout to complete
INFO[0018]   Waiting for Deployment "olm/packageserver" to rollout: 1 out of 2 new replicas have been updated
INFO[0023]   Waiting for Deployment "olm/packageserver" to rollout: 1 old replicas are pending termination
INFO[0030]   Deployment "olm/packageserver" successfully rolled out
INFO[0030] Successfully installed OLM version "latest"  

NAME                                            NAMESPACE    KIND                        STATUS
clusterserviceversions.operators.coreos.com                  CustomResourceDefinition    Installed
installplans.operators.coreos.com                            CustomResourceDefinition    Installed
subscriptions.operators.coreos.com                           CustomResourceDefinition    Installed
catalogsources.operators.coreos.com                          CustomResourceDefinition    Installed
operatorgroups.operators.coreos.com                          CustomResourceDefinition    Installed
olm                                                          Namespace                   Installed
operators                                                    Namespace                   Installed
olm-operator-serviceaccount                     olm          ServiceAccount              Installed
system:controller:operator-lifecycle-manager                 ClusterRole                 Installed
olm-operator-binding-olm                                     ClusterRoleBinding          Installed
olm-operator                                    olm          Deployment                  Installed
catalog-operator                                olm          Deployment                  Installed
aggregate-olm-edit                                           ClusterRole                 Installed
aggregate-olm-view                                           ClusterRole                 Installed
global-operators                                operators    OperatorGroup               Installed
olm-operators                                   olm          OperatorGroup               Installed
packageserver                                   olm          ClusterServiceVersion       Installed
operatorhubio-catalog                           olm          CatalogSource               Installed

# Check OLM's status.
$ operator-sdk olm status
INFO[0000] Fetching CRDs for version "0.14.1"           
INFO[0002] Fetching resources for version "0.14.1"      
INFO[0002] Successfully got OLM status for version "0.14.1"

NAME                                            NAMESPACE    KIND                        STATUS
olm                                                          Namespace                   Installed
operatorgroups.operators.coreos.com                          CustomResourceDefinition    Installed
catalogsources.operators.coreos.com                          CustomResourceDefinition    Installed
subscriptions.operators.coreos.com                           CustomResourceDefinition    Installed
installplans.operators.coreos.com                            CustomResourceDefinition    Installed
aggregate-olm-edit                                           ClusterRole                 Installed
catalog-operator                                olm          Deployment                  Installed
olm-operator                                    olm          Deployment                  Installed
operatorhubio-catalog                           olm          CatalogSource               Installed
olm-operators                                   olm          OperatorGroup               Installed
aggregate-olm-view                                           ClusterRole                 Installed
operators                                                    Namespace                   Installed
global-operators                                operators    OperatorGroup               Installed
olm-operator-serviceaccount                     olm          ServiceAccount              Installed
packageserver                                   olm          ClusterServiceVersion       Installed
system:controller:operator-lifecycle-manager                 ClusterRole                 Installed
clusterserviceversions.operators.coreos.com                  CustomResourceDefinition    Installed
olm-operator-binding-olm                                     ClusterRoleBinding          Installed
```

**Note:** certain cluster types may already have OLM enabld, but under a non-default (`"olm"`) namespace.
Set `--olm-namespac=[non-default-olm-namespace]` for both `operator-sdk olm` and `operator-sdk run`
commands if this is the case.

### Simple Operator

Assuming everything your Operator needs to successfully deploy and run exists
in your bundle directory (no `Secret`'s, etc.), you can deploy your Operator
using the following command invocation:

```console
$ operator-sdk run --olm --manifests deploy/olm-catalog-memcached-operator --operator-version 0.1.0
INFO[0000] loading Bundles                               dir=deploy/olm-catalog/memcached-operator
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=memcached-operator load=bundles
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=0.1.0 load=bundles
INFO[0000] found csv, loading bundle                     dir=deploy/olm-catalog/memcached-operator file=memcached-operator.v0.1.0.clusterserviceversion.yaml load=bundles
INFO[0000] loading bundle file                           dir=deploy/olm-catalog/memcached-operator/0.1.0 file=example.com_memcacheds_crd.yaml load=bundle
INFO[0000] loading bundle file                           dir=deploy/olm-catalog/memcached-operator/0.1.0 file=memcached-operator.v0.1.0.clusterserviceversion.yaml load=bundle
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=metadata load=bundles
INFO[0000] loading Packages and Entries                  dir=deploy/olm-catalog/memcached-operator
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=memcached-operator load=package
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=0.1.0 load=package
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=metadata load=package
INFO[0000] Creating registry                            
INFO[0000]   Creating ConfigMap "olm/memcached-operator-registry-bundles"
INFO[0000]   Creating Deployment "olm/memcached-operator-registry-server"
INFO[0000]   Creating Service "olm/memcached-operator-registry-server"
INFO[0000] Waiting for Deployment "olm/memcached-operator-registry-server" rollout to complete
INFO[0000]   Waiting for Deployment "olm/memcached-operator-registry-server" to rollout: 0 out of 1 new replicas have been updated
INFO[0001]   Waiting for Deployment "olm/memcached-operator-registry-server" to rollout: 0 of 1 updated replicas are available
INFO[0002]   Deployment "olm/memcached-operator-registry-server" successfully rolled out
INFO[0002] Creating resources                           
INFO[0002]   Creating CatalogSource "default/memcached-operator-ocs"
INFO[0002]   Creating Subscription "default/memcached-operator-v0-0-1-sub"
INFO[0002]   Creating OperatorGroup "default/operator-sdk-og"
INFO[0002] Waiting for ClusterServiceVersion "default/memcached-operator.v0.1.0" to reach 'Succeeded' phase
INFO[0002]   Waiting for ClusterServiceVersion "default/memcached-operator.v0.1.0" to appear
INFO[0034]   Found ClusterServiceVersion "default/memcached-operator.v0.1.0" phase: Pending
INFO[0035]   Found ClusterServiceVersion "default/memcached-operator.v0.1.0" phase: InstallReady
INFO[0036]   Found ClusterServiceVersion "default/memcached-operator.v0.1.0" phase: Installing
INFO[0036]   Found ClusterServiceVersion "default/memcached-operator.v0.1.0" phase: Succeeded
INFO[0037] Successfully installed "memcached-operator.v0.1.0" on OLM version "0.14.1"

NAME                            NAMESPACE    KIND                        STATUS
memcacheds.cache.example.com    default      CustomResourceDefinition    Installed
memcached-operator.v0.1.0       default      ClusterServiceVersion       Installed
```

This command can be further simplified by passing only `--operator-version=0.1.0`
and letting the command infer the bundle path.

### Multi-namespaced/cluster-wide deployment

Lets say you modify your CSV's `installModes` to support `MultiNamespace`. Now
you can deploy your Operator such that it watches multiple namespaces `ns-1`, `ns-2`, and `ns-3`:

```console
$ operator-sdk run --olm --operator-version 0.1.0 --install-mode-MultiNamespace=ns-1,ns-2,ns-3
```

### Supplying your own catalog manifests

I now have a reasonably complicated set of catalog requirements, ex. multiple `OperatorGroup`'s in my cluster,
and want full control over how the SDK tells OLM to deploy my operator.

These manifests are in `deploy/catalog-manifests`:

```console
$ ls -l deploy/catalog-manifests
-rw-rw-r--. 1 user user  2368 Mar 24 19:21 special-operatorgroup.yaml
-rw-rw-r--. 1 user user  3264 Mar 24 19:23 special-subscription.yaml
-rw-rw-r--. 1 user user  5117 Mar 24 18:22 special-catalogsource.yaml
```

Instead of `operator-sdk run --olm` creating these manifests for my Operator,
I can direct the command to create the above manifests:

```console
$ operator-sdk run --olm --operator-version 0.1.0 --include special-operatorgroup.yaml,special-subscription.yaml,special-catalogsource.yaml
INFO[0000] loading Bundles                               dir=deploy/olm-catalog/memcached-operator
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=memcached-operator load=bundles
...
INFO[0002] Creating resources                           
INFO[0002]   Creating OperatorGroup "default/special-operatorgroup"
INFO[0002]   Creating CatalogSource "default/special-subscription"
INFO[0002]   Creating Subscription "default/special-catalogsource"
...
```

### Cleaning up

I was testing my Operator, and now I would like to remove it from your cluster
in a given namespace. The `operator-sdk cleanup --olm` command will do this for you:

```console
$ operator-sdk cleanup --olm --operator-version 0.1.0
INFO[0000] loading Bundles                               dir=deploy/olm-catalog/memcached-operator
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=memcached-operator load=bundles
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=0.1.0 load=bundles
INFO[0000] found csv, loading bundle                     dir=deploy/olm-catalog/memcached-operator file=memcached-operator.v0.1.0.clusterserviceversion.yaml load=bundles
INFO[0000] loading bundle file                           dir=deploy/olm-catalog/memcached-operator/0.1.0 file=example.com_memcacheds_crd.yaml load=bundle
INFO[0000] loading bundle file                           dir=deploy/olm-catalog/memcached-operator/0.1.0 file=memcached-operator.v0.1.0.clusterserviceversion.yaml load=bundle
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=metadata load=bundles
INFO[0000] loading Packages and Entries                  dir=deploy/olm-catalog/memcached-operator
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=memcached-operator load=package
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=0.1.0 load=package
INFO[0000] directory                                     dir=deploy/olm-catalog/memcached-operator file=metadata load=package
INFO[0000] Deleting resources                           
INFO[0000]   Deleting CatalogSource "default/memcached-operator-ocs"
INFO[0000]   Deleting Subscription "default/memcached-operator-v0-0-1-sub"
INFO[0000]   Deleting OperatorGroup "default/operator-sdk-og"
INFO[0000]   Deleting CustomResourceDefinition "default/memcacheds.example.com"
INFO[0000]   Deleting ClusterServiceVersion "default/memcached-operator.v0.1.0"
INFO[0000]   Waiting for deleted resources to disappear
INFO[0001] Successfully uninstalled "memcached-operator.v0.1.0" on OLM version "0.14.1"
```

## Caveats

- `operator-sdk run` (and `operator-sdk cleanup`) are intended to be used for testing
purposes only as of now, since this command creates a transient image registry that
should not be used in production. Typically a registry is deployed eparately and a
set of catalog manifests are created in the cluster to inform OLM
of that registry and which Operator versions it can deploy and where to deploy the Operator.
- `operator-sdk run` can only deploy one Operator and one version of that Operator
at a time, hence its intended purpose being testing only. Production OLM usage
requires the typical setup described above.


[olm]:https://github.com/operator-framework/operator-lifecycle-manager/
[csv]:https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/building-your-csv.md
[pkg-manifest]:https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/how-to-update-operators.md
[operator-bundle]:https://github.com/operator-framework/operator-registry/tree/v1.5.3#manifest-format
[sdk-olm-design]:https://github.com/operator-framework/operator-sdk/blob/master/doc/proposals/sdk-integration-with-olm.md
[cli-olm-install]:https://github.com/operator-framework/operator-sdk/blob/master/doc/cli/operator-sdk_olm_install.md
[cli-olm-status]:https://github.com/operator-framework/operator-sdk/blob/master/doc/cli/operator-sdk_olm_status.md
[cli-generate-csv]:https://github.com/operator-framework/operator-sdk/blob/master/doc/cli/operator-sdk_generate_csv.md
[sdk-release-v0.15.0]:https://github.com/operator-framework/operator-sdk/releases/tag/v0.15.0
[sdk-user-guide-go]:https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md
[sdk-user-guide-ansible]:https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md
[sdk-user-guide-helm]:https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md
