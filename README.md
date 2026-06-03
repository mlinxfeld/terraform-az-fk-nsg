# terraform-az-fk-nsg

This repository contains a reusable **Terraform / OpenTofu module** and
progressive examples for defining and attaching **Azure Network Security Groups (NSGs)**
to **Subnets** (and integrating with NIC-level attach via compute modules) in a clean, explicit, and
architecture-aware way.

It is part of the **[FoggyKitchen.com training ecosystem](https://foggykitchen.com/courses/azure-fundamentals-terraform-course/)** and is designed as a dedicated **network security boundary layer** for Azure workloads.

This module is also part of the **[Azure Fundamentals with Terraform/OpenTofu — Build Real-World Azure Architectures with Reusable Modules (2026 Edition)](https://foggykitchen.com/courses/azure-fundamentals-terraform-course/)** course. In the training, it is used to explain how Azure traffic filtering is modeled at subnet and NIC level in practical, reusable infrastructure code.

Support expectations are documented in [SUPPORT.md](SUPPORT.md).

---

## Used By

This module is used as a building block by the higher-level [FoggyKitchen Landing Zone Orchestrator](https://github.com/foggykitchen/foggykitchen-landing-zone-orchestrator), where it is composed into Azure, OCI, and multicloud landing zone patterns.

## 🎯 Purpose

The goal of this repository is to provide a **clear, educational, and composable
reference implementation** for **Azure Network Security Groups** using
Infrastructure as Code.

It focuses on:

- NSGs as **first-class security boundaries** in Azure networking
- Explicit modeling of **inbound and outbound rules**
- Clear distinction between **NIC-level** (workload-scoped) and **subnet-level** (tier-scoped) security
- Clean integration with **VM-based compute** and **Load Balancer–based fan-in**
- Terraform/OpenTofu patterns that reflect **Azure’s real traffic evaluation model**

This is **not** a landing zone, platform framework, or full security stack.
It is a **learning-first building block** designed to integrate cleanly with
other FoggyKitchen modules.

---

## ✨ What the module does

Depending on configuration and example used, the module can:

- Create an **Azure Network Security Group (NSG)**
- Define **inbound and outbound security rules**
- Attach NSGs to **Subnets (tier-level security)**
- Support multiple rule definitions with priorities and source scoping
- Cleanly separate **security policy** from compute and networking resources

The module intentionally does **not** create or manage:

- Virtual Networks or subnets (handled by `terraform-az-fk-vnet`)
- Virtual Machines or VM Scale Sets (handled by `terraform-az-fk-compute`)
- Azure Bastion (handled by dedicated networking modules/examples)
- Azure Load Balancers (handled by `terraform-az-fk-loadbalancer`)
- Azure Firewall / NVAs
- Application Gateway / Front Door
- Identity, policy, or compliance frameworks

Each of those concerns belongs in its own dedicated module.

---

## 📂 Repository Structure

```bash
terraform-az-fk-nsg/
├── examples/
│   ├── 01_nic_level_nsg_private_access/
│   ├── 02_subnet_level_nsg_with_lb/
│   └── README.md
├── main.tf
├── inputs.tf
├── outputs.tf
├── versions.tf
├── LICENSE
└── README.md
```

---

## 🚀 Example Usage

### NIC-level NSG (workload-scoped)

```hcl
module "vm_nsg" {
  source = "git::https://github.com/foggykitchen/terraform-az-fk-nsg.git?ref=v1.0.0"

  name                = "fk-private-vm-nsg"
  location            = "westeurope"
  resource_group_name = "fk-rg"

  rules = [
    {
      name                       = "allow-ssh-from-bastion"
      priority                   = 100
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = "22"
      source_address_prefix      = "10.10.100.0/26"
      destination_address_prefix = "*"
      description                = "Allow SSH only from AzureBastionSubnet."
    }
  ]

  tags = {
    project = "foggykitchen"
    env     = "dev"
  }
}

# NIC-level attach is configured via compute module.

module "compute" { 
  source = "github.com/mlinxfeld/terraform-az-fk-compute"

  (...)

  attach_nsg_to_nic = true
  nsg_id            = module.vm_nsg.id

  (...)
}

```

### Subnet-level NSG (tier-scoped)

```hcl
module "private_subnet_nsg" {
  source = "git::https://github.com/foggykitchen/terraform-az-fk-nsg.git?ref=v1.0.0"

  name                = "fk-backend-subnet-nsg"
  location            = "westeurope"
  resource_group_name = "fk-rg"

  rules = [
    {
      name                       = "allow-http-from-azure-lb"
      priority                   = 100
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = "80"
      source_address_prefix      = "AzureLoadBalancer"
      destination_address_prefix = "*"
      description                = "Allow HTTP only from Azure Load Balancer."
    }
  ]

  subnet_associations = {
    private_subnet = {
      subnet_id = module.vnet.subnet_ids["fk-subnet-private"]
    }
  }

  tags = {
    project = "foggykitchen"
    env     = "dev"
  }
}
```

---

## 📤 Outputs

| Output | Description |
|--------|-------------|
| `id` | ID of the created Network Security Group |
| `name` | Name of the NSG |
| `resource_group_name` | Resource group name where NSG was created |
| `subnet_association_ids` | IDs of subnet-NSG association resources (if any) |

---

## 🧠 Design Philosophy

- Security boundaries must be **explicit**
- Attachment scope (**NIC vs subnet**) is a **first-order architectural decision**
- NSG rules should be **minimal, intentional, and source-scoped**
- One module = one responsibility
- Networking and security should be modeled the way Azure **actually evaluates traffic**
- Compute, load balancing, and security are **separate concerns**

This repository intentionally avoids abstractions that hide NSG mechanics
behind “magic” defaults.

---

## 🧩 Related Modules & Training

- [terraform-az-fk-vnet](https://github.com/foggykitchen/terraform-az-fk-vnet)  
- [terraform-az-fk-compute](https://github.com/mlinxfeld/terraform-az-fk-compute)  
- [terraform-az-fk-loadbalancer](https://github.com/foggykitchen/terraform-az-fk-loadbalancer)  
- [terraform-az-fk-bastion](https://github.com/mlinxfeld/terraform-az-fk-bastion)  
- [terraform-az-fk-disk](https://github.com/foggykitchen/terraform-az-fk-disk)  
- [terraform-az-fk-storage](https://github.com/foggykitchen/terraform-az-fk-storage)  
- [terraform-az-fk-aks](https://github.com/mlinxfeld/terraform-az-fk-aks)  

---

## 🪪 License

Licensed under the **Universal Permissive License (UPL), Version 1.0**.  
See [LICENSE](LICENSE) for details.

---

© 2026 [FoggyKitchen.com](https://foggykitchen.com) - Cloud. Code. Clarity.
