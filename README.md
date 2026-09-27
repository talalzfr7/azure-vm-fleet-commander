# Azure VM Fleet Commander

> A parameterised Bicep design for repeatable Azure VM provisioning in an MSP client context.

**Status: design and specification complete; implementation not yet merged.** This public repository records the intended architecture, security decisions, module contracts, and verification criteria. The Bicep implementation and Azure deployment evidence are intentionally not published yet.

## Client scenario

An MSP needs to provision a small, isolated VM environment for a client without hand-building each resource in the portal. The project is designed around a reusable orchestration template: client-specific values change in parameter files while the deployment shape, security baseline, and reviewable infrastructure code remain consistent.

The intended deployment contains:

- One resource group per client environment.
- A virtual network and workload subnet.
- A network security group with narrowly scoped inbound rules.
- A configurable number of Azure virtual machines, each with its own NIC and OS disk.
- Availability-zone placement as a parameter.
- Administrative access through the client's management path, not public IPs on workload VMs.

## Planned architecture

```text
Resource group
└── VNet (10.0.0.0/16)
    └── Workload subnet (10.0.1.0/24)
        ├── NSG: source-scoped administration + web rules
        ├── VM 0
        │   ├── NIC
        │   └── OS disk
        └── VM N
            ├── NIC
            └── OS disk
```

The project is intended to integrate with the Network Landing Zone repository for shared management services such as Azure Bastion. The current source specification names Bastion as the administrative access pattern but does not yet define a Bastion module in this repository; that boundary is recorded as an open design decision rather than silently treated as implemented.

## Planned module contract

| Module | Responsibility |
|---|---|
| `bicep/modules/network.bicep` | VNet, workload subnet, and NSG with parameterised address prefixes and source-scoped rules. |
| `bicep/modules/vm.bicep` | VM, NIC, and OS disk for one instance; returns VM ID and private IP. |
| `bicep/main.bicep` | Orchestrates the network module and a `vmCount` loop over the VM module. |
| `bicep/parameters.dev.json` | Small development sizing. |
| `bicep/parameters.prod.json` | Production sizing; values remain subject to current SKU/cost validation. |

## Security design

- Workload VMs have no public IP addresses.
- SSH and RDP rules accept a specific source address prefix, never `Any`.
- HTTP and HTTPS are permitted only where the workload requires them.
- The NSG's default deny behaviour remains part of the design; an unnecessary explicit deny-all rule must not obscure Azure's rule evaluation model.
- Administrative credentials are secure deployment inputs, not committed values.
- Bastion or an equivalent private management path is an integration dependency; exposing RDP or SSH directly to the internet is out of scope.

## Verification criteria

Implementation will not be considered complete until the following are demonstrated and documented:

1. Each module compiles with `az bicep build`.
2. The orchestration template compiles and the loop expands to the configured VM count.
3. A disposable test resource group deploys the VNet, subnet, NSG, VMs, NICs, and disks.
4. Effective security rules are inspected for at least one VM.
5. The deployment has no workload public IPs.
6. An architecture diagram and deployment evidence are added only after a real test.
7. The test resource group and other credit-consuming resources are deleted and the cleanup is verified.

## Evidence boundary

This repository currently proves that the design has been specified. It does **not** yet prove that the templates compile, deploy, or have been operated in Azure. Those claims will be added only with the corresponding code, command output, and deployment records.

## AZ-104 and MSP relevance

The project connects the AZ-104 compute/IaC domain to a realistic MSP workflow: repeatable onboarding, parameterised infrastructure, explicit network security, and a deliverable that another engineer can review and reuse.

- Sprint scope: SD-31 through SD-35.
- Exam domain: Deploy and manage Azure compute resources.
- Design references: [Azure Bicep modules](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/modules), [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/), and [Azure VM resource reference](https://learn.microsoft.com/en-us/azure/templates/microsoft.compute/virtualmachines).
- Study-plan source: [AZ-104 sprint repository](https://github.com/talalzfr7/az104-sprint).

See [`docs/adr.md`](docs/adr.md) for the decisions and [`docs/design-specification.md`](docs/design-specification.md) for the implementation contract.
