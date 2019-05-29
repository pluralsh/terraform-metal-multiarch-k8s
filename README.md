Multi-Architecture Kubernetes on Packet
==

This is a [Terraform](https://www.terraform.io/docs/providers/packet/index.html) project for deploying Kubernetes on [Packet](https://packet.com) with node pools of x86 and ARM devices. 

This project configures your cluster with:

- [MetalLB](https://metallb.universe.tf/) using Packet elastic IPs.
- [Packet CSI](https://github.com/packethost/csi-packet) storage driver.

Requirements
-

The only required variables are `auth_token` (your [Packet API](https://www.packet.com/developers/api/#) key), `count_x86` (the number of x86 devices), and `count_arm` (ARM devices). 

Other options include `secrets_encryption` (`"yes"` configures your controller with encryption for secrets--this is disabled by default), and fields like `facility` (the Packet location to deploy to) and `plan_x86` or `plan_arm` (to determine the server type of these architectures) can be specified as well. Refer to `vars.tf` for a complete catalog of tunable options.

Node Pool Management
-

To instantiate a new node pool **after initial spinup**, in `3-kube-node.tf1`, define a pool using the node pool module like this:

```hcl
module "node_pool_green" {
  source = "modules/node_pool"

  kube_token         = "${module.kube_token_2.token}"
  kubernetes_version = "${var.kubernetes_version}"
  pool_label         = "green"
  count_x86          = "${var.count_x86}"
  count_arm          = "${var.count_arm}"
  plan_x86           = "${var.plan_x86}"
  plan_arm           = "${var.plan_arm}"
  facility           = "${var.facility}"
  cluster_name       = "${var.cluster_name}"
  controller_address = "${packet_device.k8s_primary.network.0.address}"
  project_id         = "${packet_project.kubernetes_multiarch.id}"
}
```
where the label is `green` (rather than the initial pool, `blue`) and then, generate a new `kube_token` (ensure the module name matches the `kube_token` field in the spec above, i.e. `kube_token_2`) by defining this in `1-provider.tf` (or anywhere before the node_pool instantiation):

```hcl
module "kube_token_2" {
  source = "modules/kube-token"
}
```
Generate your new token:
```
terraform apply -target=module.kube_token_2
```
On your controller, [add your new token](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/#cmd-token-create), and then apply the new node pool:
```
terraform apply -target=module.node_pool_green
```
At which point, you can either destroy the old pool, or taint/evict pods, etc. once this new pool connects.
