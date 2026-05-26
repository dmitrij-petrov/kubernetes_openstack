# kubernetes_openstack

Ansible and kubeadm workflow for deploying Kubernetes on OpenStack.

This repository is for operators who want a reproducible way to stand up and tear down Kubernetes clusters on top of OpenStack infrastructure using automation instead of manual control-plane assembly.

## What it does

- Provisions cluster infrastructure in OpenStack
- Bootstraps a Kubernetes control plane with `kubeadm`
- Adds worker nodes
- Supports cluster teardown
- Includes certificate update workflow

## Deployment model

The workflow assumes OpenStack credentials, an existing SSH keypair, and a small set of environment-specific inputs such as image, network, flavors, node counts, and storage options. The automation is designed to cover both basic clusters and more customized OpenStack-backed deployments.

## Example use case

A platform team needs to create a short-lived Kubernetes environment in OpenStack for testing or service rollout validation. Instead of manually provisioning instances, configuring networking, and initializing the cluster by hand, this repository provides one deployment workflow from infrastructure to working cluster.

## Getting started

The following mandatory environment variables need to be set before calling `ansible-playbook`:

  * `OS_*`: standard OpenStack environment variables such as `OS_AUTH_URL`, `OS_USERNAME`, ...
  * `KEY`: name of an existing SSH keypair

The following optional environment variables can also be set:

  * `NAME`: name of the Kubernetes cluster, used to derive instance names, `kubectl` configuration and security group name
  * `IMAGE`: name of an existing Ubuntu 16.04 image
  * `EXTERNAL_NETWORK`: name of the neutron external network, defaults to 'public'
  * `FLOATING_IP_POOL`: name of the floating IP pool
  * `FLOATING_IP_NETWORK_UUID`: uuid of the floating IP network (required for LBaaSv2)
  * `USE_OCTAVIA`: try to use Octavia instead of Neutron LBaaS, defaults to False
  * `USE_LOADBALANCER`: assume a loadbalancer is used and allow traffic to nodes (default: false)
  * `SUBNET_CIDR` the subnet CIDR for OpenStack's network (default: `10.8.10.0/24`)
  * `POD_SUBNET_CIDR` CIDR of the POD network (default: `10.96.0.0/16`)
  * `CLUSTER_DNS_IP`: IP address of the cluster DNS service passed to kubelet (default: `10.96.0.10`)
  * `BLOCK_STORAGE_VERSION`: version of the block storage (Cinder) service, defaults to 'v2'
  * `IGNORE_VOLUME_AZ`: whether to ignore the AZ field of volumes, needed on some clouds where AZs confuse the driver, defaults to False.
  * `NODE_MEMORY`: how many MB of memory should nodes have, defaults to 4GB
  * `NODE_FLAVOR`: allows to configure the exact OpenStack flavor name or ID to use for the nodes. When set, the `NODE_MEMORY` setting is ignored.
  * `NODE_COUNT`: how many nodes should we provision, defaults to 3
  * `NODE_AUTO_IP` assign a floating IP to nodes, defaults to False
  * `NODE_DELETE_FIP`: delete floating IP when node is destroyed, defaults to True
  * `NODE_BOOT_FROM_VOLUME`: boot node instances using boot from volume. Useful on clouds with only boot from volume
  * `NODE_TERMINATE_VOLUME`: delete the root volume when each node instance is destroy, defaults to True
  * `NODE_VOLUME_SIZE`: size of each node volume. defaults to 64GB
  * `NODE_EXTRA_VOLUME`: create an extra unmounted data volume for each node, defaults to False
  * `NODE_EXTRA_VOLUME_SIZE`: size of extra data volume for each node, defaults to 80GB
  * `NODE_DELETE_EXTRA_VOLUME`: delete the extra data volume for each node when node is destroy, defaults to True
  * `MASTER_BOOT_FROM_VOLUME`: boot the master instance on a volume for data persistence, defaults to True
  * `MASTER_TERMINATE_VOLUME`: delete the volume when master instance is destroy, defaults to True
  * `MASTER_VOLUME_SIZE`: size of the master volume. default to 64GB
  * `MASTER_MEMORY`: how many MB of memory should master have, defaults to 4 GB
  * `MASTER_FLAVOR`: allows to configure the exact OpenStack flavor name or ID to use for the master. When set, the `MASTER_MEMORY` setting is ignored.
  * `INCLUDE_HELM`: installs helm to the cluster, defaults to False
  * `HELM_REPOS`: a list of additional helm repos to add, separated by semicolons. Example: `charts* https://github.com/helm/charts;mycharts https://github.com/dev/mycharts`
  * `HELM_INSTALL`: a list of helm charts and their parameters to install, separated by semicolons. Example: `mycharts/mychart;charts/somechart --name somechart --namespace somenamespace`

Spin up a new cluster:

```console
$ ansible-playbook site.yaml -e os_project_name={{ os_project_name }}
```

Destroy the cluster:

```console
$ ansible-playbook destroy.yaml -e os_project_name={{ os_project_name  }}
```

```console
$ ansible-playbook update_certs.yaml -e os_project_name={{ os_project_name }}
```

* `os_project_name == tenant in openstack`


## Open Issues

### Find a better way to configure worker nodes' network plugin

Somehow, the network plugin (kubenet) is not correctly set on the worker node. On the master node `/var/lib/kubelet/kubeadm-flags.env` (created by `kubeadm init`) contains: 

```bash
KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd --cloud-provider=external --network-plugin=kubenet --pod-infra-container-image=k8s.gcr.io/pause:3.1 --resolv-conf=/run/systemd/resolve/resolv.conf"
```

It contains the correct `--network-plugin=kubenet` as configured [here](https://github.com/pfisterer/k8s-on-openstack-wip-k8s-1.15/blob/master/files/kubeadm-init.yaml.j2#L9). After joining the k8s cluster, the worker node's copy of `/var/lib/kubelet/kubeadm-flags.env` (created by `kubeadm join`) looks like this: 

```bash
KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --resolv-conf=/run/systemd/resolve/resolv.conf"
```

It contains `--network-plugin=cni` despite setting `network-plugin: kubenet` [here](https://github.com/pfisterer/k8s-on-openstack-wip-k8s-1.15/blob/master/files/kubeadm-init.yaml.j2#L21). But the JoinConfiguration is ignored by `kubeadm join` when using a join token. 

Once I edit `/var/lib/kubelet/kubeadm-flags.env` to contain --network-plugin=kubenet, the worker node goes online. I've added a hack in [roles/kubeadm-nodes/tasks/main.yaml](https://github.com/pfisterer/k8s-on-openstack-wip-k8s-1.15/blob/master/roles/kubeadm-nodes/tasks/main.yaml#L12) to set the correct value.


## Prerequisites

  * Ansible (tested with version 2.8.3)
  * Shade library required by Ansible OpenStack modules (`python-shade` for Debian)

## CI/CD

The following environment variables needs to be defined:

  * `OS_AUTH_URL`
  * `OS_PASSWORD`
  * `OS_USERNAME`
  * `OS_DOMAIN_NAME`

# References

  * https://kubernetes.io/docs/getting-started-guides/kubeadm/
