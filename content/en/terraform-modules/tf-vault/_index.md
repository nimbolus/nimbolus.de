---
title: "Terraform Vault"
linkTitle: "Terraform Vault"
date: "2021-06-03T00:23:09+02:00"
weight: 2
description: "Provision a HA Vault cluster with Consul storage and auto unseal."

github_project_repo: https://github.com/nimbolus/tf-vault
---
[View on GitHub](https://github.com/nimbolus/tf-vault.git)

These Terraform modules can provision a highly available Vault cluster with the Consul storage backend.

You can use the [vault-consul](https://github.com/nimbolus/tf-vault/tree/master/vault-consul) module to deploy the Vault cluster using a pre-existing Consul installation.
The [vault-transit](https://github.com/nimbolus/tf-vault/tree/master/vault-transit) module can be used to setup the transit engine used for auto-unsealing another Vault cluster (they rely on each other for auto-unsealing).

## Usage
1. Deploy Vault 1 (e.g. with vault-consul) - do NOT enable auto-unseal yet.
2. Repeat for Vault 2.
3. Deploy vault-transit for Vault 1 and Vault 2.
4. Enable auto-unseal for Vault 1 and init seal migration. You need to restart the Vault server pods.
5. Repeat for Vault 2.

## Examples

### HA Vault with Consul storage
[View on GitHub](https://github.com/nimbolus/tf-vault/tree/master/examples/vault-auto-unseal/main.tf)

```terraform
module "vault1" {
  source                            = "github.com/nimbolus/tf-vault/vault-consul"
  release_name                      = "vault1"
  namespace                         = kubernetes_namespace.vault.metadata[0].name
  vault_domain                      = yamldecode(kubectl_manifest.vault1_certificate.yaml_body_parsed)["spec"]["dnsNames"][0]
  consul_address                    = "consul.example.com"
  auto_unseal_enable                = true
  auto_unseal_vault_service_address = "https://vault2.example.com"
  auto_unseal_vault_mount_path      = "vault-transit"
  auto_unseal_vault_role            = "vault-unseal"
  tls_secret_name                   = yamldecode(kubectl_manifest.vault1_certificate.yaml_body_parsed)["spec"]["secretName"]
  ingress_ssl_passthrough_enable    = true
}
```

### Transit Secret Engine for Auto Unseal
```terraform
module "vault_transit1" {
  depends_on = [module.vault1]
  providers = {
    vault = vault.vault1
   }
  source    = "github.com/nimbolus/tf-vault/vault-transit"
  name      = "vault2"
  namespace = "vault2"
}
```
