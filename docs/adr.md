# Architecture Decision Record: VM Fleet Commander

- **Status:** Proposed design baseline
- **Date:** 2026-09-27
- **Scope:** Repeatable MSP client VM provisioning
- **Implementation status:** Not yet published or deployed

## Context

An MSP needs a repeatable way to provision a small client workload environment. The operator should be able to change the client name, VM count, sizing, address prefixes, and environment values without rewriting the deployment. The result must be reviewable as infrastructure code and must not make public exposure the default administrative path.

## Decision 1: Bicep over hand-authored ARM JSON

Bicep is the implementation language because it is the Azure-native declarative language used by the sprint and produces an ARM deployment without the verbosity of hand-authored JSON. It supports typed parameters, modules, loops, outputs, and compile-time validation while remaining close to Azure resource semantics. The compiled ARM output remains a verification artefact, not the authoring surface.

This serves operational excellence: the template should be understandable to the engineer who reviews or adapts it for the next client.

## Decision 2: Parameterised modules over one monolithic template

The design separates network concerns from one-VM concerns. The network module owns the VNet, subnet, and NSG. The VM module owns one VM instance, its NIC, and its OS disk. The orchestration template composes them and uses a loop for the requested count.

This gives the MSP a reusable unit of change. A correction to the VM definition should not require duplicating the network definition, and a change to the network baseline should not require editing every VM declaration. Module outputs make the dependency explicit: the network module supplies the subnet and NSG context; the VM module returns the VM ID and private IP for later reporting.

## Decision 3: Planned resource set

The first implementation is expected to create:

- A resource group supplied by the deployment target.
- One VNet with a parameterised address space.
- One workload subnet with a parameterised prefix.
- One NSG with source-scoped SSH/RDP rules and workload HTTP/HTTPS rules.
- One NIC and one OS disk per VM.
- A configurable number of VMs, with a default development count of two.

Azure Bastion is the intended administrative access pattern, but the current source specification does not define a Bastion module in this repository. The present decision is to treat Bastion as an integration dependency supplied by the Network Landing Zone rather than claim that this repository deploys it. That boundary must be resolved before implementation is described as complete.

## Decision 4: Security baseline

Workload VMs have no public IP addresses. Administrative access is intended to traverse Bastion or another approved private management path. This reduces the exposed attack surface, but it also creates an operational dependency: the management path must be available and tested.

The NSG permits SSH and RDP only from a parameterised source address prefix, not from `Any`. HTTP and HTTPS are allowed only for the workload's intended service surface. The design relies on Azure's default deny behaviour for other inbound traffic; an explicit deny-all rule is not required and could distract from the priority and direction model the project is meant to teach.

Administrative credentials are passed as secure deployment inputs. No password, token, or connection string belongs in a parameter file committed to this repository.

## Decision 5: Environment parameterisation

The orchestration template exposes `vmCount`, `environmentName`, VM sizing, administrative inputs, and network values. The planned development profile uses two smaller VMs; the production profile uses a larger VM size. Exact SKU availability and cost must be checked at implementation time rather than treated as permanent facts.

## Client deliverable

When implemented, a client-facing delivery should contain:

1. The versioned Bicep module set.
2. Environment parameter guidance without secret values.
3. An architecture diagram.
4. Deployment and teardown instructions.
5. The ADR explaining the security and operational trade-offs.
6. Deployment evidence and any deviations discovered during testing.

## Risks and open decisions

- **Bastion boundary:** decide whether the fleet repository consumes a shared Bastion from the landing zone or receives an explicit management-subnet contract.
- **Source prefix:** define how a client supplies and rotates the approved administrative source range.
- **Image and OS baseline:** select and document a supported image before implementation.
- **Cost:** validate VM SKU, disk, and any management-resource costs against the active Azure offer before deployment.
- **Evidence:** do not add deployment-success language until a real disposable deployment has been tested and cleaned up.
