---
title: "Terraform K3s"
linkTitle: "Terraform K3s"
date: "2021-05-11T01:41:54+02:00"
weight: 1
description: "Provision a HA K3s cluster on OpenStack, Hetzner or anything else."

github_project_repo: https://github.com/nimbolus/tf-k3s
---
[View on GitHub](https://github.com/nimbolus/tf-k3s.git)

These Terraform modules can provision k3s nodes and are able to build a cluster from multiple nodes.

You can use the [k3s](https://github.com/nimbolus/tf-k3s/tree/master/k3s) module to template the necessary cloudinit files for creating a k3s cluster node.
Modules for [OpenStack](https://github.com/nimbolus/tf-k3s/tree/master/k3s-openstack) and [Hetzner hcloud](https://github.com/nimbolus/tf-k3s/tree/master/k3s-hcloud) that bundle all necessary resources are available.

## Examples

Note that network, subnet and key pair needs to be created beforehand.

### Simple Module for templating cloudinit user_data
[View on GitHub](https://github.com/nimbolus/tf-k3s/tree/master/examples/basic/main.tf)

```terraform
resource "random_password" "cluster_token" {
  length  = 64
  special = false
}

module "k3s_master" {
  source = "github.com/nimbolus/tf-k3s/k3s"

  name             = "k3s-master"
  k3s_token        = random_password.cluster_token.result
  install_k3s_exec = "server --disable traefik --node-label az=ex1"
}

module "k3s_worker" {
  source = "github.com/nimbolus/tf-k3s/k3s"

  name              = "k3s-worker"
  k3s_join_existing = true
  k3s_url           = module.k3s_master.k3s_url
  k3s_token         = random_password.cluster_token.result
  install_k3s_exec  = "agent --node-label az=ex1"
}
```

### HA-Master and bootstrap token with OpenStack
[View on GitHub](https://github.com/nimbolus/tf-k3s/tree/master/examples/ha-openstack/main.tf)

```terraform
resource "random_password" "ha_cluster_token" {
  length  = 64
  special = false
}

resource "random_password" "ha_bootstrap_token_id" {
  length  = 6
  special = false
}

resource "random_password" "ha_bootstrap_token_secret" {
  length  = 16
  special = false
}

module "master1" {
  source = "github.com/nimbolus/tf-k3s/k3s-openstack"

  name              = "k3s-master1"
  image_name        = "ubuntu-20.04"
  flavor_name       = "m1.small"
  availability_zone = "ex1"
  keypair_name      = "example-keypair"
  network_id        = "example-id"
  subnet_id         = "example-id"

  k3s_token              = random_password.ha_cluster_token.result
  install_k3s_exec       = "server --cluster-init --kube-apiserver-arg=\"enable-bootstrap-token-auth\" --node-label az=ex1"
  bootstrap_token_id     = nonsensitive(random_password.ha_bootstrap_token_id.result)
  bootstrap_token_secret = random_password.ha_bootstrap_token_secret.result
}

output "k3s_url" {
  value = module.master1.k3s_url
}

resource "local_file" "kubeconfig" {
  filename = "kubeconfig.yaml"
  content  = module.master1.kubeconfig
}

module "master2" {
  source = "github.com/nimbolus/tf-k3s/k3s-openstack"

  name              = "k3s-master2"
  image_name        = "ubuntu-20.04"
  flavor_name       = "m1.small"
  availability_zone = "ex1"
  keypair_name      = "example-keypair"
  network_id        = "example-id"
  subnet_id         = "example-id"
  security_group_id = module.master1.security_group_id

  k3s_join_existing = true
  k3s_url           = module.master1.k3s_url
  k3s_token         = random_password.ha_cluster_token.result
  install_k3s_exec  = "server --kube-apiserver-arg=\"enable-bootstrap-token-auth\" --node-label az=ex1"
}

module "master3" {
  source = "github.com/nimbolus/tf-k3s/k3s-openstack"

  name              = "k3s-master3"
  image_name        = "ubuntu-20.04"
  flavor_name       = "m1.small"
  availability_zone = "ex1"
  keypair_name      = "example-keypair"
  network_id        = "example-id"
  subnet_id         = "example-id"
  security_group_id = module.master1.security_group_id

  k3s_join_existing = true
  k3s_url           = module.master1.k3s_url
  k3s_token         = random_password.ha_cluster_token.result
  install_k3s_exec  = "server --kube-apiserver-arg=\"enable-bootstrap-token-auth\" --node-label az=ex1"
}

module "worker1" {
  source = "github.com/nimbolus/tf-k3s/k3s-openstack"

  name              = "k3s-worker1"
  image_name        = "ubuntu-20.04"
  flavor_name       = "m1.small"
  availability_zone = "ex1"
  keypair_name      = "example-keypair"
  network_id        = "example-id"
  subnet_id         = "example-id"
  security_group_id = module.master1.security_group_id

  k3s_join_existing = true
  k3s_url           = module.master1.k3s_url
  k3s_token         = random_password.ha_cluster_token.result
  install_k3s_exec  = "agent --node-label az=ex1"
}

provider "kubernetes" {
  host                   = module.master1.k3s_url
  token                  = "${module.master1.bootstrap_token_id}.${module.master1.bootstrap_token_secret}"
  cluster_ca_certificate = base64decode(module.master1.ca_crt)
}
```

### Single Master and Worker with hcloud
[View on GitHub](https://github.com/nimbolus/tf-k3s/tree/master/examples/basic-hcloud/main.tf)

```terraform
resource "random_password" "hcloud_cluster_token" {
  length  = 64
  special = false
}

module "hcloud_master" {
  source = "github.com/nimbolus/tf-k3s/k3s-hcloud"

  name         = "k3s-master"
  keypair_name = "pubkey"
  network_id   = "example-id"

  k3s_token        = random_password.hcloud_cluster_token.result
  install_k3s_exec = "server --disable traefik --node-label az=ex1"
}

module "hcloud_worker" {
  source = "github.com/nimbolus/tf-k3s/k3s-hcloud"

  name         = "k3s-worker"
  keypair_name = hcloud_ssh_key.yubikey.name
  network_id   = hcloud_network.k3s.id

  k3s_join_existing = true
  k3s_url           = module.hcloud_master.k3s_url
  k3s_token         = random_password.hcloud_cluster_token.result
  install_k3s_exec  = "agent --node-label az=ex1"
}

resource "local_file" "hcloud_kubeconfig" {
  filename = "hcloud-kubeconfig.yaml"
  content  = module.hcloud_master.kubeconfig
}
```
