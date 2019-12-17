# Dynamic provisioner for local volumes

**Warning**: this project is not actively maintained anymore. We recommend the following alternatives:

* [TopoLVM](https://github.com/cybozu-go/topolvm) goes beyond what this project attempts to achieve. It dynamically provisions LVM-based volumes, implements the CSI interface, while accounting for each host total storage capacity.
* [Sig-storage local static provisioner](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) automatically creates static PersistentVolumes based on the devices available to each host.
* [OpenEBS](https://github.com/openebs/openebs) is a CNCF container attached storage ([CAS](https://www.cncf.io/blog/2018/04/19/container-attached-storage-a-primer/)) solution for Kubernetes. OpenEBS provides pluggable architecture with different flavors of storage engines, one specially targets dynamic provisioning of local PV [OpenEBS LocalPV](https://docs.openebs.io/docs/next/localpv.html) with both hostpath, device mapping options as well as configuring local PVs backed by ZFS ([ZFS LocalPV](https://github.com/openebs/zfs-localpv)).

**Important**: this is an experimental project, not intended for production use cases. There is no public Docker image release available.

This projects allows the creation of PersistentVolumeClaim (PVC) for local volumes.
A PersistentVolume (PV) corresponding to this PVC will dynamically be created.
A filesystem volume will dynamically be created and mounted on the host where the pod is scheduled.

## Components

This project is composed of 3 main components:

* The provisioner: watches new PVC resources and creates matching PV resources on the apiserver. Cluster-wide deployment of a single pod.
* The driver client: binary deployed on k8s nodes that respects k8s flex interface. Called by kubelet when a scheduled pod needs a local persistent volume to be mounted. Contacts the driver daemon through a unix socket.
* The driver daemon: daemonset with one pod per k8s node. Listens to calls from the client. Creates a logical volume for the pod on the underlying host (eg. /mnt/storage/my-pvc), and bind mount it within the pod directory on the underlying host (eg. /var/lib/kubelet/pods/pod-id/volumes/elastic-local/my-pvc).

## Development

To start, get a working development Kubernetes cluster using [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/#install-minikube) or [GKE](https://cloud.google.com/kubernetes-engine/):

```bash
make -C ../operators bootstrap-minikube
# or
GCLOUD_PROJECT=my-project-id make -C ../operators bootstrap-gke
```

If you chose Minikube then attach a new disk to the virtual machine:

```bash
make minikube-attach-disk
```

Finally initialize a logical volume group:

```bash
make minikube-create-vg # or gke-create-vg
```

## Usage

Build and push the Docker image:

```bash
make docker-build-push
```

Deploy on Kubernetes:

```bash
make deploy
```

Then you can create any PVC matching the `elastic-local` storage class and Pod using this PVC. Example:

```bash
kubectl apply -f config/pvc-sample.yaml -f config/pod-sample.yaml
```

## Architecture

![architecture](https://github.com/elastic/local-volume/blob/master/architecture.svg)

The provisioner only interacts with the APIServer: it watches any new PVC matching our StorageClass provisioner, and dynamically creates a matching PV.

When the pod gets scheduled on a host, the kubelet calls our driver binary in order to mount a volume in the given pod directory. In turn, our driver binary calls our driver daemon (HTTP through a unix domain socket). The driver daemons is responsible for creating the volume, creating an actual directory mount for it on the file system, then bind-mounting this directory into the given pod directory.

## Limitations

* Thinly provisioned volumes are not supported on GKE
* We don't handle total storage capacity requirements.

## Flex spec

We are doing implementation option 1 from the spec below :)

```
----------- Mandatory ------------
<driver executable> init // Performs driver initialisation
----------- Implementation option 1 -----------
<driver executable> attach <json options> <node name> // Attaching persistent volume to the host
<driver executable> waitforattach <mount device> <json options> // Waiting for volume to be attached (10 minutes timeout)
<driver executable> isattached <json options> <node name> // Check if the volume is attached
<driver executable> detach <mount device> <node name> // Detaching persistent volume from the host
<driver executable> mountdevice <mount dir> <mount device> <json options> // Mount the device to a global path so the pod can bind to it by Kubelet
<driver executable> unmountdevice <mount device> // Un-mount the device from global path
----------- Implementation option 2-----------
<driver executable> mount <mount dir> <json options> // This call-out implements both attach and mountdevice functions
<driver executable> unmount <mount dir> // This call-out implements both unmountdevice and detach functions
```
