# CUDN EVPN Kubernetes Manager: VM-to-Fabric Networking

| Field | Value |
|---|---|
| Author(s) | Benny Kopilov |
| Jira | [OSAC-4291](https://redhat.atlassian.net/browse/OSAC-4291) |
| Alignment work | [OSAC-6758](https://redhat.atlassian.net/browse/OSAC-6758), under [OSAC-6757](https://redhat.atlassian.net/browse/OSAC-6757) |
| Date | 2026-10-25 |

This revision follows the approved [networking contract in PR #357](https://github.com/osac-project/enhancement-proposals/pull/357) and its Unified Networking and VMaaS requirements, with the user-requested route-target compatibility amendment from [osac PR #1971](https://github.com/osac-project/osac/pull/1971) and the shared namespace lifecycle proposed for shared-contract review. Technical decisions and superseded clarifications are recorded in [clarifications](clarifications.md); implementation and evidence are described in the [design](design.md). These proposals do not change PR #357's approval status.

## Problem Statement

OSAC has an EVPN manager that creates and deletes an overlay network using Fabric-provided segment identifiers. The current manager does not implement VM SecurityGroup enforcement or the required policy binding before VM creation. Tenants need VMs and physical hosts to share an isolated network with consistent address allocation, DNS and security policy. Administrators need the existing manager to follow the approved networking contract so it can interoperate with a conforming Fabric Manager. Until that implementation and its lab validation are complete, a successfully created network alone does not establish workload connectivity or isolation.

## In Scope

- VMaaS networking on the single provider-owned networking hub, in connected IPv4 deployments, through the release's supported EVPN path.
- Fabric interoperability using the selected compatible Fabric Manager, with automatic segment coordination and disjoint VM, physical-host and guest-ingress address allocations. The host compatibility fix applies to this manager with every selected Fabric; administrators must verify matching route exchange for that pair. Real Netris interoperability is the initial required lab validation; other pairs need their own evidence.
- One VM-capable Subnet per VirtualNetwork under the current manager limit. Other Subnets remain available for physical workloads; VM attachment on one of those Subnets reports the limitation explicitly. Adding a Subnet does not invalidate the existing owner's VM network.
- Workload addresses, gateway and agreed DNS resolvers supplied through DHCP. A VM keeps its address across restart and live migration. The Fabric supplies routing to physical workloads and other Subnets in the same VirtualNetwork.
- SecurityGroup enforcement before a VM first uses its tenant network, baseline protection for unbound workloads, consistent rule updates, deterministic retries and cleanup of owned resources.
- SecurityGroups can be prepared before any Subnet exists and remain available while their VirtualNetwork exists, including after its VM-capable Subnet is removed. Removing one VM or Subnet preserves other live groups and workloads. [User]
- Administrator installation and diagnostic guidance, actionable prerequisite and unsupported-path errors, and evidence that distinguishes implementation checks from real connectivity and policy validation.

## Out of Scope

- Additional networking hubs, cross-hub connectivity, secondary tenant attachments, IPv6, dual-stack and disconnected networking deployments.
- VM attachment to multiple Subnets of one VirtualNetwork while the current one-network limit applies; an owner network is not handed to an existing sibling Subnet when its owner is deleted.
- Automatic configuration of physical links, node tunnel endpoints or BGP peering by a tenant networking task.
- New inter-VirtualNetwork peering, new network types, provider-written Kubernetes Managers, migration/adoption of existing provider state, and MetalLB address-pool provisioning by this manager.

## User Stories

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want to select the supported EVPN Kubernetes Manager with a compatible Fabric Manager during installation so that VMs can use the provider's tenant networks.
- As a Cloud Infrastructure Admin, I want explicit prerequisites for peering, address allocation and DNS so that a deployment can be prepared before tenant traffic starts.
- As a Cloud Infrastructure Admin, I want diagnostics that distinguish a missing prerequisite, a manager limitation and a connectivity failure so that the correct configuration or provider component can be repaired.

### Tenant Admin / Tenant User

- As a Tenant Admin or Tenant User, I want a VM on a supported Subnet to communicate with permitted physical workloads using the normal OSAC networking resources so that applications can span virtual and physical infrastructure.
- As a Tenant Admin or Tenant User, I want VM and physical addresses to be allocated without collisions so that both can share the Subnet reliably.
- As a Tenant Admin or Tenant User, I want selected SecurityGroups to protect VM traffic from first boot so that unbound or incompletely prepared workloads do not exchange application traffic.
- As a Tenant Admin or Tenant User, I want a VM's address to survive restart and live migration so that permitted Fabric traffic continues to reach it.
- As a Tenant Admin or Tenant User, I want an unsupported VM attachment to explain the owner-Subnet limitation while preserving existing workloads so that the network can be structured within the supported boundary.
- As a Tenant Admin or Tenant User, I want deleting my VM or network to remove its owned networking state while preserving other live resources so that retries and partial failures do not damage another workload.

## Assumptions

- The approved EVPN gateway, route-advertisement and DHCP behavior can be realized by the selected hub and Fabric versions. A real Netris/Fabric-pair result is still required; document approval does not establish runtime support.
- Supported hub APIs can provide the required DNS settings, address stability and policy behavior. The design records the unresolved mechanism choices and their implementation impact.

## Dependencies

- The selected shared namespace/policy direction requires review of the VN/Subnet lifecycle amendment, safe deletion, exact platform enforcement and concurrent policy updates before activation. [User]

- Shared-contract and Fabric-owner review must reconcile PR #1971's compatibility behavior with PR #357. Later integration with controller-provided route-target values needs an agreed API/handoff before it can replace the compatibility behavior; it is not claimed available by this revision.
- [OSAC-6766](https://redhat.atlassian.net/browse/OSAC-6766) supplies shared manager resolution, stored address layouts, ordered tasks, typed handoffs and workload integration; [OSAC-6768](https://redhat.atlassian.net/browse/OSAC-6768) tracks the source API work.
- [OSAC-6769](https://redhat.atlassian.net/browse/OSAC-6769) tracks installation settings and admission deployment. The Enclave plugin and Wizard must consume and validate those settings through their separately scoped work.
- [OSAC-6761](https://redhat.atlassian.net/browse/OSAC-6761) implements the manager-specific changes after this documentation and its implementation delta are reviewed.
- [Agentless alignment](https://redhat.atlassian.net/browse/OSAC-6755) and [Netris alignment](https://redhat.atlassian.net/browse/OSAC-6756) own Fabric realization. A Fabric/Kubernetes pair is supported only with recorded applicable validation.
- OVN-Kubernetes, KubeVirt, FRR-K8s and the installed underlay must satisfy the prerequisites and lifecycle checks in the design. BMaaS/CaaS use Fabric attachments and retain their own workload orchestration.

---

## Provenance

Authored: revise @ prd 0.11.5 - 1c6e8de, workspace codex/cudn-evpn-route-targets @ 38d18d944 (7 behind origin/main)
Final: revise @ prd 0.11.5 - 1c6e8de, workspace codex/osac-6761-cudn-evpn-contract @ a6aa33f0b

> Context changed between revise and revise.

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.5","ai_workflows":"1c6e8de","source_repo":"a6aa33f0b","source_repo_branch":"codex/osac-6761-cudn-evpn-contract","commits_behind_main":0,"commits_ahead_main":2,"main_ref":"main","phases":["revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":true} -->
