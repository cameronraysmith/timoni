# Bundle

Timoni bundles offer a declarative way of managing the lifecycle
of applications and their infra dependencies.

A Timoni bundle is a CUE file for defining a group of instances
together with their values and module references.

## Example

The following is an example of a Bundle that defines a Redis
master-replica cluster and a podinfo instance connected to the Redis instance.

```cue
bundle: {
	apiVersion: "v1alpha1"
	name:       "podinfo"
	instances: {
		redis: {
			module: {
				url:     "oci://ghcr.io/stefanprodan/modules/redis"
				version: "7.2.4"
			}
			namespace: "podinfo"
			values: maxmemory: 256
		}
		podinfo: {
			module: url:     "oci://ghcr.io/stefanprodan/modules/podinfo"
			module: version: "6.5.4"
			namespace: "podinfo"
			values: caching: {
				enabled:  true
				redisURL: "tcp://redis:6379"
			}
		}
	}
}
```

For the above example, Timoni performs the following actions at apply time:

- Validates that the Bundle definition is in conformance with the API version specified by `apiVersion`.
- For each instance, it fetches the module version from the registry using the `module.url`
  as the artifact repository address and the `module.version` as the artifact tag.
- Creates the Kubernetes namespaces if they don't exist.
- For each instance, it builds, validates and creates the Kubernetes resources using the specified `values`.
- The list of managed resources along with the module reference and values are stored in the cluster in a
  Kubernetes Secret, in the same namespace with the instance.
- If an instance already exists, Timoni performs a server-side apply dry-run to detect changes and
  applies only the resources with divergent state.
- If previously applied resources are missing from the current revision, these
  resources are deleted from the cluster.
- Waits for each instance's resources to become ready.

You can run this example by saving the Bundle into `podinfo.bundle.cue`.

Apply the Bundle on the cluster:

=== "command"

      ```sh
      timoni bundle apply -f podinfo.bundle.cue
      ```

=== "output"

      ```text
      applying instance redis
      pulling oci://ghcr.io/stefanprodan/modules/redis:7.2.4
      using module timoni.sh/redis version 7.2.4
      installing redis in namespace podinfo
      Namespace/podinfo created
      applying master
      ServiceAccount/podinfo/redis created
      ConfigMap/podinfo/redis created
      Service/podinfo/redis created
      Deployment/podinfo/redis-master created
      PersistentVolumeClaim/podinfo/redis-master created
      waiting for 5 resource(s) to become ready...
      resources are ready
      applying replica
      Service/podinfo/redis-readonly created
      Deployment/podinfo/redis-replica created
      waiting for 2 resource(s) to become ready...
      resources are ready
      applying instance podinfo
      pulling oci://ghcr.io/stefanprodan/modules/podinfo:6.5.4
      using module timoni.sh/podinfo version 6.5.4
      installing podinfo in namespace podinfo
      ServiceAccount/podinfo/podinfo created
      Service/podinfo/podinfo created
      Deployment/podinfo/podinfo created
      waiting for 3 resource(s) to become ready...
      resources are ready
      ```

Build the Bundle and print the resulting Kubernetes resources for all the Bundle's instances:
=== "command"

      ```sh
      timoni bundle build -f podinfo.bundle.cue
      ```

=== "output"

      ```text
      ---
      # Instance: redis
      ---
      apiVersion: v1
      kind: ServiceAccount
      metadata:
      labels:
        app.kubernetes.io/part-of: redis
        app.kubernetes.io/version: 7.2.4
      name: redis
      namespace: podinfo
      ---
      # Redis deployments omitted for brevity
      ---
      # Instance: podinfo
      ---
      apiVersion: v1
      kind: ServiceAccount
      metadata:
      labels:
        app.kubernetes.io/name: podinfo
        app.kubernetes.io/version: 6.5.4
      name: podinfo
      namespace: podinfo
      ---
      # Podinfo deployment omitted for brevity
      ```

List the managed resources from a bundle and their rollout status:

=== "command"

      ```sh
      timoni bundle status -f podinfo.bundle.cue
      ```

=== "output"

     ```text
     last applied 2024-03-03T20:21:19Z
     module oci://ghcr.io/stefanprodan/modules/redis:7.2.4
     digest: sha256:8cf531365742c7cab9628909dfe16958550853f7c994284eacad64f169f4c74a
     ServiceAccount/podinfo/redis Current Resource is current
     ConfigMap/podinfo/redis Current Resource is always ready
     Service/podinfo/redis Current Service is ready
     Service/podinfo/redis-readonly Current Service is ready
     Deployment/podinfo/redis-master Current Deployment is available. Replicas: 1
     Deployment/podinfo/redis-replica Current Deployment is available. Replicas: 1
     PersistentVolumeClaim/podinfo/redis-master Current PVC is Bound

     last applied 2024-03-03T20:21:19Z
     module oci://ghcr.io/stefanprodan/modules/podinfo:6.5.4
     digest: sha256:1dba385f9d56f9a79e5b87344bbec1502bd11f056df51834e18d3e054de39365
     ServiceAccount/podinfo/podinfo Current Resource is always ready
     Service/podinfo/podinfo Current Service is ready
     Deployment/podinfo/podinfo Current Deployment is available. Replicas: 1
     ```

List the instances in Bundle `podinfo` across all namespaces:

=== "command"

      ```sh
      timoni list --bundle podinfo -A
      ```

=== "output"

     ```text
     NAME    NAMESPACE         MODULE                                          VERSION LAST APPLIED          BUNDLE
     podinfo podinfo           oci://ghcr.io/stefanprodan/modules/podinfo      6.5.4   2024-03-03T16:20:07Z  podinfo
     redis   podinfo           oci://ghcr.io/stefanprodan/modules/redis        7.2.4   2024-03-03T16:20:00Z  podinfo
     ```

## Writing a Bundle spec

A Bundle file must contain a definition that matches the following schema:

```cue
#Bundle: {
	apiVersion: string
	name:       string
	instances: [string]: {
		module: {
			url:     string
			digest?: string
			version: *"latest" | string
		}
		namespace: string
		values: {...}
	}
}
```

Bundle files can contain arithmetic
[operations](https://cuelang.org/docs/tutorials/tour/expressions/operators/),
[string interpolation](https://cuelang.org/docs/tutorials/tour/expressions/interpolation/)
and everything else that CUE [std lib](https://cuelang.org/docs/references/spec/) supports.

### API version

The `apiVersion` is a required field that specifies the version of the Bundle schema.

Currently, the only supported value is `v1alpha1`.

### Name

The `name` is a required field used to track the ownership of instances deployed to a Kubernetes cluster.

Note that Bundles should have unique names per cluster, using the same name for different bundles
will result in [ownership conflict](#transfer-ownership).

### Instances

The `instances` array is a required field that specifies the list of Instances part of this Bundle.

A Bundle must contain at least one instance with the following required fields:

```cue
bundle: {
	apiVersion: "v1alpha1"
	name:       "podinfo"
	instances: {
		podinfo: {
			module: url: "oci://ghcr.io/stefanprodan/modules/podinfo"
			namespace: "podinfo"
		}
	}
}
```

### Instance Module

The `instance.module` is a required field that specifies the OCI URL, version and/or digest
of the instance's module.

#### URL

The `instance.module.url` is a required field that specifies the source of the module.
It can be either an OCI repository address (preferred) or a local path to a module (useful during development).

When using an OCI repository, the `url` field must be in the format `oci://<registry-host>/<repo-name>`.

When using a Local path, the `url` field must be in the format `file://path/to/module`.

!!! tip Relative paths

    Relative paths are always computed relatively to the path of the bundle file containing the value.

#### Version

The `instance.module.version` is an optional field that specifies the version number of the module.
The version number must follow [Timoni's semantic versioning](cue/module/publishing.md#version-format).
When not specified, the version defaults to `latest`, which pulls the module OCI artifact tagged as latest.

```cue
module: {
	url:     "oci://ghcr.io/stefanprodan/modules/podinfo"
	version: "6.5.4"
}
```

!!! tip "Default version"

    When not specified, the version defaults to `latest`, which pulls the module OCI artifact tagged as latest.
    Note that using `version: "latest"` is not recommended for production system, unless you also specify a digest.  

#### Digest

The `instance.module.digest` is an optional field that specifies the OCI digest of the module.

```cue
module: {
	url:    "oci://ghcr.io/stefanprodan/modules/podinfo"
	digest: "sha256:1dba385f9d56f9a79e5b87344bbec1502bd11f056df51834e18d3e054de39365"
}
```

When both the version number and the digest are specified, Timoni will verify that the
upstream digest of the version matches the specified `instance.module.digest`.

```cue
module: {
	url:     "oci://ghcr.io/stefanprodan/modules/podinfo"
	version: "6.5.4"
	digest:  "sha256:1dba385f9d56f9a79e5b87344bbec1502bd11f056df51834e18d3e054de39365"
}
```

If the version is set to `latest` and a digest is specified, Timoni will ignore the version
and will pull the module by its OCI digest.

### Instance Namespace

The `instance.namespace` is a required field that specifies the Kubernetes namespace where the instance is created.

If the specified namespace does not exist, Timoni will first create the namespace,
then it will apply the instance's resources in that namespace.

### Instance Values

The `instance.values` is an optional field that specifies custom values used to configure the instance.

At apply time, Timoni merges the custom values with the defaults,
validates the final values against the config schema and creates the instance.

#### Values from runtime

The `@timoni(runtime:[string|number|bool]:[VAR_NAME])` CUE attribute can be placed next
to a field to set its value from the [Runtime](bundle-runtime.md).

```cue
values: {
	host:    "example.com" @timoni(runtime:string:MY_HOST)
	enabled: true          @timoni(runtime:bool:MY_ENABLED)
	score:   1             @timoni(runtime:number:MY_SCORE)
}
```

To make a Runtime attribute required, the field value can be set to its type:

```cue
values: {
	host:    string @timoni(runtime:string:MY_HOST)
	enabled: bool   @timoni(runtime:bool:MY_ENABLED)
	score:   int    @timoni(runtime:number:MY_SCORE)
}
```

The Runtime values can come from Kubernetes API and/or from the environment variables,
for more details please see the [Bundle Runtime documentation](bundle-runtime.md).

## Working with Bundles

### Install and Upgrade

To install or upgrade the instances defined in a Bundle file,
you can use the `timoni bundle apply` command.

Example:

```shell
timoni bundle apply -f bundle.cue
```

The apply command performs the following actions for each instance:

- Pulls the module version from the specified container registry.
- If the registry is private, uses the credentials found in `~/.docker/config.json`.
- If the registry credentials are specified with `--creds`, these take priority over the docker ones.
- Merges the custom values supplied in the Bundle with the default values found in the module.
- Builds the module by passing the instance name, namespace and values.
- Labels the resulting Kubernetes resources with the instance name and namespace.
- Creates the instance namespace if it doesn't exist.
- Applies the Kubernetes resources on the cluster.
- Creates or updates the instance inventory with the last applied resources IDs.

### Diff Upgrade

After editing a bundle file, you can review the changes that will
be made on the cluster with `timoni bundle apply --diff`.

Example:

```shell
timoni bundle apply --dry-run --diff -f bundle.cue
```

### Force Upgrade

If an upgrade contains changes to immutable fields, such as changing the image
tag of a Kubernetes Job, you need to set the `--force` flag.

Example:

```shell
timoni bundle apply --force -f bundle.cue
```

With `--force`, Timoni will recreate only the resources that contain changes
to immutable fields.

### Transfer ownership

If an install or upgrade involves Instances already created, either separately or as a part of another Bundle,
the operation will fail.
To transfer ownership to the current Bundle, you need to set the `--overwrite-ownership` flag.

Example:

```shell
timoni bundle apply --overwrite-ownership -f bundle.cue
```

### Status

To list the current status of the managed resources for each
instance including the last applied date, the module url and digest,
you can use the `timoni bundle status`.

Example using the bundle name:

```shell
timoni bundle status my-bundle
```

Example using a bundle CUE file:

```shell
timoni bundle status -f bundle.cue
```

### Build

To build the instances defined in a Bundle file and print the resulting Kubernetes resources,
you can use the `timoni bundle build` command.

Example:

```shell
timoni bundle build -f bundle.cue
```

### Use values from JSON and YAML files

A bundle can be defined in multiple files of different formats:

```shell
timoni bundle build -f bundle.cue -f extras1.json -f extras2.yaml
```

Timoni extracts the CUE values from the JSON and YAML files,
and unifies them with the bundle value. Note that setting the
same field in multiple files is not supported.

Timoni supports the following extensions: `.cue`, `.json`, `.yml`, `.yaml`.

### Uninstall

To uninstall all the instances belonging to a Bundle,
you can use the `timoni bundle delete` command.

Example using the bundle name:

```shell
timoni bundle delete my-bundle
```

Example using a bundle CUE file:

```shell
timoni bundle delete -f bundle.cue
```

Timoni will search the cluster and delete all the instances having
the `bundle.timoni.sh/name: <name>` label matching the given bundle name.
The instances are uninstalled in reverse order,
first created instance is last to be deleted.

### Garbage collection

Timoni's garbage collector keeps track of the applied resources and prunes the Kubernetes
objects that were previously applied but are missing from the current revision.

Example:

```shell
timoni bundle apply -f bundle.cue
```

To prevent the garbage collector from deleting certain
resources such as Kubernetes Persistent Volumes,
these resources can be annotated with `action.timoni.sh/prune: "disabled"`.

### Readiness checks

By default, Timoni applies the instances in order, and will wait for
each instance's resources to become ready, before moving to the next instance.

The readiness check is performed for the Kubernetes resources with the following types:

- Kubernetes built-in kinds: Deployment, DaemonSet, StatefulSet,
  PersistentVolumeClaim, Pod, PodDisruptionBudget, Job, CronJob, Service,
  Secret, ConfigMap, CustomResourceDefinition
- Custom resources that are compatible
  with [kstatus](https://github.com/kubernetes-sigs/cli-utils/tree/master/pkg/kstatus)

Example:

```shell
timoni bundle apply --wait --timeout=5m -f bundle.cue
```

With `--timeout`, Timoni will retry the readiness checks until the specified
timeout period expires. If an instance's resource fails to become ready,
the apply command will exit with an error.

The readiness check is enabled by default, to opt-out set `--wait=false`.

### Vetting

To verify that one or more CUE files contain a valid Bundle definition,
you can use the `timoni bundle vet` command.

Example:

```shell
timoni bundle vet -f bundle.cue -f extras.cue
```

If the validation passes, Timoni will list all the instances found in the computed bundle.

When `--print-value` is specified, Timoni will write the Bundle computed value to stdout.

Example:

```shell
timoni bundle vet -f bundle.cue --runtime-from-env --print-value
```

Printing the computed value is particular useful when debugging runtime attributes.

### Format

To format Bundle files, you can use the `cue fmt` command.

Example:

```shell
cue fmt bundle.cue
```

### Referencing local modules

When developing and testing Timoni Modules, you can reference them
from a Bundle file using relative local paths.

Example repo structure:

```shell
├── modules
│   ├── app1
│   └── app2
└── bundles
    └── apps-test.cue
```

Example Bundle file:

```cue
bundle: {
	apiVersion: "v1alpha1"
	name:       "apps-test"
	instances: {
		"app1": {
			module: url: "file://../modules/app1"
			namespace: "app1"
			values: {...}
		}
		"app2": {
			module: url: "file://../modules/app2"
			namespace: "app2"
			values: {...}
		}
	}
}
```

When using local paths, the `url` field must be in the format `file://path/to/module`
and the module path is computed relatively to the path of the bundle file location.

Note that when using local modules, the module's version and digest are ignored, as these
are only relevant when pulling modules from a container registry.
All instances created from modules referenced with local paths have
the module version set to `0.0.0-devel`.
