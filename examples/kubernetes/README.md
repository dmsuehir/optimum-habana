# Deploy Optimum for Intel® Gaudi® Accelerators Examples to Kubernetes

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 1.11.1](https://img.shields.io/badge/AppVersion-1.11.1-informational?style=flat-square)

This folder contains a Dockerfile and [Helm chart](https://helm.sh) demonstrating how to run 🤗 Optimum Habana examples
can be run using nodes with Intel® Gaudi® AI accelerator from a vanilla Kubernetes cluster. The instructions below
explain how to build the docker image and deploy the job to a Kubernetes cluster using Helm.

## Requirements

### Client System Requirements

In order to build the docker images and deploy a job to Kubernetes you will need:

* [Docker](https://docs.docker.com/engine/install/)
* [Docker compose](https://docs.docker.com/compose/install/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/) installed and configured to access a cluster
* [Helm CLI](https://helm.sh/docs/intro/install/)

### Cluster Requirements

Your Kubernetes cluster will need the [Intel Gaudi Device Plugin for Kubernetes](https://docs.habana.ai/en/latest/Orchestration/Gaudi_Kubernetes/Device_Plugin_for_Kubernetes.html) in order to request and utilize the accelerators
in the Kubernetes job. Also, ensure that your Kubernetes version is supported based on the
[support matrix](https://docs.habana.ai/en/latest/Support_Matrix/Support_Matrix.html#support-matrix).

## Container

The [Dockerfile](Dockerfile) and [docker-compose.yaml](docker-compose.yaml) builds the following images:

* An `optimum-habana` base image that uses the [PyTorch Docker images for Gaudi](https://developer.habana.ai/catalog/pytorch-container/) as it's base, then installs optimum-habana and Deep Speed.
* A `optimum-habana-examples` image is built on top of the `optimum-habana` base to includes installations from
`requirements.txt` files in the example directories and a clone of [this GitHub repository](https://github.com/huggingface/optimum-habana/) in order to run example scripts.

To build the containers, clone this repository, and then use the following commands:

```bash
cd examples/kubernetes

# Set variables for your container registry and repository
export REGISTRY=<Your container registry>
export REPO=<Your container repository>

# Specify the base image name/tag
export BASE_IMAGE_NAME=vault.habana.ai/gaudi-docker/1.15.1/ubuntu22.04/habanalabs/pytorch-installer-2.2.0
export BASE_IMAGE_TAG=latest

# Specify your Gaudi Software version and Optimum Habana version
export GAUDI_SW_VER=1.15.1
export OPTIMUM_HABANA_VER=1.11.1

# Build the images
docker compose build
```

After the build has completed, push the `optimum-habana-examples` image to your registry so that it can be pulled from
the Kubernetes cluster.

## Helm Chart

### Kubernetes resources

This Kubernetes job uses a [Helm chart](https://helm.sh) with the following resources:
* Job to run the Optimum Habana example script using HPU(s)
* [Persistant volume claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
  (PVC) backed by NFS for output files
* (Optional) Pod to access the PVC files after the example script completes
* (Optional) [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) for a Hugging Face token, if gated
  models are being used

### Helm chart values

The [Helm chart values file](https://helm.sh/docs/chart_template_guide/values_files/) is a yaml file with values that
get passed to the chart when it's deployed to the cluster. These values specify the parameters for your job, the
name and tag of your Docker image, the number of HPU cards to use for the job, etc.

<details>
  <summary> Expand to see the values table </summary>

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | Optionally provide node [affinities](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity) to constrain which node your worker pod will be scheduled on |
| command | list | `[]` |  |
| env | list | `[{"name":"LOGLEVEL","value":"info"}]` | Define environment variables to set in the container |
| envFrom | list | `[{"configMapRef":{"name":"intel-proxy-config"}}]` | Define a config map's data as container environment variables |
| image.pullPolicy | string | `"IfNotPresent"` | Determines when the kubelet will pull the image to the worker nodes. Choose from: `IfNotPresent`, `Always`, or `Never`. If updates to the image have been made, use `Always` to ensure the newest image is used. |
| image.repository | string | `nil` | Repository and name of the docker image |
| image.tag | string | `nil` | Tag of the docker image |
| imagePullSecrets | list | `[]` |  |
| nodeSelector | object | `{}` | Optionally specify a [node selector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) with labels the determine which node your worker pod will land on |
| podAnnotations | object | `{}` | Pod [annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) to attach metadata to the job |
| podSecurityContext | object | `{}` | Specify a pod security context to run as a non-root user |
| resources.limits."habana.ai/gaudi" | int | `1` | Specify the number of Gaudi card(s) |
| resources.limits.cpu | int | `16` | Specify [CPU resource](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu) limits for the job |
| resources.limits.hugepages-2Mi | string | `"4400Mi"` | Specify hugepages-2Mi requests for the job |
| resources.limits.memory | string | `"128Gi"` | Specify [Memory limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory) requests for the job |
| resources.requests."habana.ai/gaudi" | int | `1` | Specify the number of Gaudi card(s) |
| resources.requests.cpu | int | `16` | Specify [CPU resource](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu) requests for the job |
| resources.requests.hugepages-2Mi | string | `"4400Mi"` | Specify hugepages-2Mi requests for the job |
| resources.requests.memory | string | `"128Gi"` | Specify [Memory resource](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory) requests for the job |
| secret.encodedToken | string | `nil` | Hugging Face token encoded using base64. |
| secret.secretMountPath | string | `"/tmp/hf_token"` | If a token is provided, specify a mount path that will be used to set HF_TOKEN_PATH |
| securityContext.privileged | bool | `true` | Run as privileged or unprivileged |
| storage.accessModes | list | `["ReadWriteMany"]` | [Access modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) for the persistent volume. |
| storage.deployDataAccessPod | bool | `true` | A data access pod will be deployed when set to true |
| storage.pvcMountPath | string | `"/tmp/pvc-mount"` | Locaton where the PVC will be mounted in the pods |
| storage.resources | object | `{"requests":{"storage":"30Gi"}}` | Storage [resources](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#resources) |
| storage.storageClassName | string | `"nfs-client"` | Name of the storage class to use for the persistent volume claim. To list the available storage classes use: `kubectl get storageclass`. |
| tolerations | list | `[]` | Optionally specify [tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) to allow the worker pod to land on a node with a taint. |

</details>

### Deploy job to the cluster

After updating the values file for the example that you want to run, use the following command to deploy the job to
your Kubernetes cluster.

```bash
cd examples/kubernetes

helm install -f <path to values.yaml> optimum-habana-examples . -n <namespace>
```

After the job is deployed, you can check the status of the pods:
```bash
kubectl get pods -n <namespace>
```

To monitor a running job, you can view the logs of the worker pod:

```bash
kubectl logs <pod name> -n <namespace> -f
```

The data access pod can be used to copy artifacts from the persistent volume claim (for example, the trained model
after fine tuning completes). Note that this requires that the data access pod be deployed to the cluster with the helm
chart by setting `storage.deployDataAccessPod = true` in the values yaml file. The path to the files is defined in the
`storage.pvcMountPath` value (this defaults to `/tmp/pvc-mount`). You can find the name of your data access pod using
`kubectl get pods -n <namespace> | grep dataaccess`.

```bash
# Copy files from the PVC mount path to your local machine
kubectl cp <data access pod name>:/tmp/pvc-mount . -n <namespace>
```

Finally, when your job is complete and you've copied all the artifacts that you need to your local machine, you can
uninstall the helm job from the cluster:

```bash
helm uninstall optimum-habana-examples . -n <namespace>
```

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.13.1](https://github.com/norwoodj/helm-docs/releases/v1.13.1)
