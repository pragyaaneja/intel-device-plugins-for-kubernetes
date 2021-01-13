## Intel SGX Plugin for AKS
The [SGX plugin](cmd/sgx_plugin/README.md) is responsible for discovering and reporting SGX
device nodes to `kubelet`. Instead of using the complete Intel plugin solution, due to its bulkiness, we deploy the SGX plugin as a daemonset directly. This daemonset first runs an init container which uses the [SGX resource manager](https://github.com/microsoft/sgx-resource-manager/tree/main) to advertise the SGX EPC memory for ACC nodes as a kubernetes extended resource named `sgx.intel.com/epc`. Then it continues with the container image for the SGX device plugin, which advertises two unitless device node access resources, namely `sgx.intel.com/enclave` and `sgx.intel.com/provision`.

| Resource | Meaning |
|:---- |:-------- |
| `sgx.intel.com/enclave` | the number of containers per node allowed to use `/dev/sgx/enclave` (default limit: `110`) |
| `sgx.intel.com/provision` | the number of containers per node allowed to use `/dev/sgx/provision` (default limit: `110`) |
| `sgx.intel.com/epc` | the amount of EPC memory available on each node in bytes |

## Installation
#### Build the plugin image
The given Makefile will use `docker` to build a local container image called `intel/intel-sgx-plugin`with the tag `devel`. The image name including the docker username and teh tag can be changed in the [Makefile](/Makefile#L127-L130).
```bash
$ cd ${INTEL_DEVICE_PLUGINS_SRC}
$ make intel-sgx-plugin
...
Successfully tagged <docker-image:tag>
$ docker push <docker-image:tag>
...
xxxxxxxxxx: Pushed
```
#### Deploy the daemonset
Deploying the plugin involves the deployment of the ServiceAccount and ClusterRole required for SGX extended resource manager to advertise and access node resources, in addition to deployment of the daemonset in [SGX DaemonSet YAML](/deployments/sgx_plugin/base/intel-sgx-plugin.yaml) with the necessary configuration, including the the built and tagged docker image. The YAML file can be deployed as follows: 
```bash
$ cd ${INTEL_DEVICE_PLUGINS_SRC}
$ kubectl apply -f ${INTEL_DEVICE_PLUGINS_SRC}/deployments/sgx_plugin/base/role.yml
...
serviceaccount/sgx-extres-api created
clusterrole.rbac.authorization.k8s.io/sgx-extres-reader created
clusterrolebinding.rbac.authorization.k8s.io/sgx-extres-reader created

$ kubectl apply -f ${INTEL_DEVICE_PLUGINS_SRC}/deployments/sgx_plugin/base/intel-sgx-plugin.yaml
...
daemonset.apps/intel-sgx-plugin created
```
#### Verify SGX device plugin is registered on master
Verification of the plugin deployment and detection of SGX hardware can be confirmed by examining the resource allocations on the nodes:
```bash
$ kubectl describe node <node name> | grep sgx.intel.com
 sgx.intel.com/enclave:    110
 sgx.intel.com/epc:        56623104
 sgx.intel.com/provision:  110
 sgx.intel.com/enclave:    110
 sgx.intel.com/epc:        56623104
 sgx.intel.com/provision:  110
 sgx.intel.com/enclave    0           0
 sgx.intel.com/epc        0           0
 sgx.intel.com/provision  0           0
```