# Design Specification: VM Fleet Commander

## Inputs

| Input | Type | Intent |
|---|---|---|
| `vmCount` | `int` | Number of VM module instances; planned default: `2`. |
| `environmentName` | `string` | Environment discriminator such as `dev` or `prod`. |
| `vmSize` | `string` | VM size selected for the environment. |
| `adminUsername` | `string` | Administrative account name subject to image/OS constraints. |
| `adminPassword` | `securestring` | Deployment-time secret; never committed. |
| `vnetName` | `string` | Network resource name. |
| `addressPrefix` | `string` | VNet address space; planned default `10.0.0.0/16`. |
| `subnetName` | `string` | Workload subnet name. |
| `subnetPrefix` | `string` | Workload subnet prefix; planned default `10.0.1.0/24`. |
| `nsgName` | `string` | NSG resource name. |
| `adminSourcePrefix` | `string` | Approved source range for SSH/RDP; must not default to `*`. |
| `availabilityZone` | `string` | Planned default zone for the VM instances. |

The current two-VM profiles use one zone by default. The implementation must not describe this as zone redundancy; spreading instances across zones is a separate design choice tied to the client availability requirement.

## Module interfaces

### `network.bicep`

Creates the VNet, workload subnet, and NSG. The NSG rules must be explicit about direction, protocol, destination port, source prefix, access, and priority. Outputs must include the subnet resource ID and NSG resource ID so the parent template can pass them to VM instances.

### `vm.bicep`

Creates one VM, one NIC, and one OS disk. It receives the subnet and NSG context from the parent. Outputs must include the VM resource ID and private IP address. The module must not create a public IP.

### `main.bicep`

Deploys the network module first, then creates `vmCount` VM module instances using a Bicep loop. The generated resource names must be deterministic from the client/environment inputs and the loop index.

## Planned repository shape

```text
bicep/
├── main.bicep
├── modules/
│   ├── network.bicep
│   └── vm.bicep
├── parameters.dev.json
└── parameters.prod.json
architecture/
├── deployment-screenshot.png       # after a real deployment
└── vm-fleet-architecture.png       # after diagram review
docs/
├── adr.md
└── design-specification.md
```

## Acceptance matrix

| Area | Acceptance evidence |
|---|---|
| Compilation | `az bicep build` succeeds for both modules and `main.bicep`. |
| Loop | Compiled output contains the configured number of VM module instances. |
| Network | VNet and subnet use the selected address spaces; NSG priorities and source ranges are inspectable. |
| Security | No VM public IP; administrative rules are not open to `Any`. |
| Deployment | Disposable resource group contains the expected resources. |
| Operations | Deployment, verification, and teardown steps are documented. |
| Honesty | Observed results are separated from planned behaviour. |
