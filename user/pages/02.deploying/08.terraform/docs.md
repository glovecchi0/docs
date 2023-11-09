---
title: Terraform
taxonomy:
    category: docs
---

### Deploy Using Terraform

Using the Terraform resource [helm\_release](https://registry.terraform.io/providers/hashicorp/helm/latest/docs/resources/release) you can install NeuVector during the provisioning phase of the Kubernetes cluster (Day 1), managed (AKS, EKS, GKE, IBM…), or on Virtual Machines in the same way.

This would allow NeuVector to follow the infrastructure lifecycle, ensuring the K8s cluster is protected from its creation as a basic service.

Take a look [here](https://developer.hashicorp.com/terraform/intro) to understand why you should use an Infrastructure as Code (IaC) tool like Terraform to build (Day 1) and operate (Day 2) your infrastructure.

Following this model, it will be possible to create the infrastructure on the Cloud Provider or Hypervisor that you use in your Company, install Kubernetes, if you don't use managed clusters, and have the Security Platform in a single step and using a single tool.

##### Fully IaC high-level example

- Infrastructure   - [Harvester](https://github.com/rancherlabs/harvester-equinix-terraform/blob/main/README.md)
- Virtual Machines - [Harvester](https://registry.terraform.io/modules/terraform-harvester-modules/vm/harvester/latest)
- Kubernetes       - [RKE2](https://registry.terraform.io/modules/rancher/rke2-install/null/latest)
- NeuVector        - helm\_release - A detailed explanation will follow below

### NeuVector TF files

#### Directory structure

```
$ tree tf-projects/neuvector/
.
├── README.md
├── values.yaml
├── main.tf
├── outputs.tf
├── provider.tf
├── terraform.tfvars
└── variables.tf
```

##### Simple custom values.yaml file

```
controller:
  replicas: 3
  secret:
    enabled: true
    data:
      userinitcfg.yaml: 
        always_reload: true
        users:
        -
          Fullname: admin
          Password: 
          Role: admin

manager:
  svc:
    type: LoadBalancer

cve:
  scanner:
    replicas: 2

resources:
  limits:
    cpu: 400m
    memory: 2792Mi
  requests:
    cpu: 100m
    memory: 2280Mi

containerd:
  enabled: true
  path: /var/run/containerd/containerd.sock
```

#### Terraform [Provider Configuration](https://developer.hashicorp.com/terraform/language/providers/configuration)

```
terraform {
  required_providers {
    <YOUR\_INFRASTRUCTURE\_PROVIDER>

    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.0.0"
    }

    helm = {
      source  = "hashicorp/helm"
      version = ">= 2.10.1"
    }
  }

  required_version = ">= 0.14"
}
 
provider "kubernetes" {
  config_path = "${path.cwd}/${var.prefix}_kube_config.yml"
}

provider "helm" {
  kubernetes {
    config_path = "${path.cwd}/${var.prefix}_kube_config.yml"
  }
}
```

#### Terraform Main file

```
<YOUR\_INFRASTRUCTURE\_RESOURCES>

resource "helm_release" "neuvector-core" {
  depends_on       = [resource.null_resource.first-setup]
  name             = "neuvector"
  repository       = "https://neuvector.github.io/neuvector-helm/"
  chart            = "core"
  create_namespace = true
  namespace        = "cattle-neuvector-system"

  values = [
    "${file("${path.cwd}/values.yaml")}"
  ]

  set {
    name  = "controller.secret.data.userinitcfg\\.yaml.users[0].Password"
    value = var.neuvector_password
  }
}

data "kubernetes_service" "neuvector-service-webui" {
  metadata {
    name      = "neuvector-service-webui"
    namespace = resource.helm_release.neuvector-core.namespace
  }
}
```

#### Terraform [Output](https://developer.hashicorp.com/terraform/cli/commands/output) file

```
output "neuvector-webui-url" {
  value       = "https://${data.kubernetes_service.neuvector-service-webui.status.0.load_balancer.0.ingress[0].ip}:8443"
  description = "NeuVector WebUI (Console) URL"
}
```

#### Terraform [.tfvars](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables) file

```
<YOUR\_INFRASTRUCTURE\_VARIABLES>
neuvector_password = "YourSecurePassword!1"
```
