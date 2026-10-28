# Clarification Log — CUDN EVPN Manager

## Status and Authority

This is the OSAC-6758 alignment revision for the existing OSAC-4291 documents. The user selected alignment with approved networking PR #357 on 2026-10-25, then directed continuation of the route-target fix discussion for osac PR #1971. The merged [manager contract](https://github.com/osac-project/enhancement-proposals/blob/a430f35da98461b5331f690332b1eebd3a3012b5/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md) and associated Unified/VMaaS designs remain the approved baseline. D23 records the proposed explicit-RT amendment and D24 the intended later API integration; neither is claimed approved in the shared contract.

The target requirements, proposed RT exception and user-selected shared namespace direction (D26) are recorded; review questions Q1–Q5 remain open. This revision is a review draft, not a claim that those questions, implementation or real-pair validation are complete. [OSAC-6761](https://redhat.atlassian.net/browse/OSAC-6761) remains separate from this documentation work.

## Current Decisions

| ID | Decision | Source |
|---|---|---|
| D11 | OSAC owns cudn_evpn; this release uses network type evpn on one provider-owned hub in a connected IPv4 deployment. Fabric Managers conform to the same typed interface. | Contract §§4.1–4.3, 4.9; Unified support boundary |
| D12 | All eight internal entry points consume the prescribed snapshots/API IDs in osac_job_vars. Successful no-output tasks publish nothing; overlay apply returns only vm_attachment.namespace. | Contract §§4.5–4.6, 4.10 |
| D13 | Core records gateway/overlay/physical/lbVip allocation once. The manager consumes that plan and reserves all CIDRs outside overlay; it does not derive a competing plan from fabric_reserved_range. | Contract §4.8 |
| D14 | The approved baseline uses a primary Layer2 EVPN CUDN, installed VTEP, supplied different integer L2/L3 VNIs and MTU 1500. Its omitted-RT/<hub-AS>:<VNI> clause is replaced in this proposal by D23. The existing shared AS/VNI admission rule is retained pending Q5 review; no additional VNI range is claimed by this exception. | Contract §§4.9–4.10; proposed D23 |
| D15 | Fabric owns the anycast gateway and routing. CUDN has no OVN router port; DHCP supplies the recorded gateway and CUDN_EVPN_DNS_RESOLVERS. Dual overlay/Fabric gateway MAC matching is not an installation task. | Contract §§4.8–4.10 |
| D16 | First create_subnet to claim an unowned VN network records the owner Subnet's API ID. Other Subnets' network tasks succeed without realization, and their overlay applies fail permanently. Owner delete removes the network and releases the claim, with no handover. | Contract §§4.7, 4.10 |
| D17 | Core serializes Subnet tasks per VN. Managers use atomic backend operations or retryable conflict detection, take no manager locks, and ensure deletion wins over cancelled late writes. | Contract §4.7 and hub-network edge cases |
| D18 | Baseline deny covers unbound pods both ways, preserving DHCP/ARP. Group policy exists without a Subnet, binds by ComputeInstance API-ID label, keeps all bound interfaces on current rules and sets both policy directions; D26 selects one shared policy object per group. Unsupported ICMP must fail permanently if not enforceable. | Contract §4.10 |
| D19 | Apply prepares a deterministic tenant-labelled primary-network namespace before VM creation and never reads or modifies the VM. VMaaS owns the VM, pod labels and l2bridge attachment. Address stability across restart/live migration is required. | Contract §4.10; VMaaS VM construction |
| D20 | Networking settings and credentials enter through the networking instance-group environment; tasks use the hub service account. Missing/malformed settings and prerequisites are retryable; unsupported paths/policy are final. Errors are sanitized and failed tasks fail the play. | Contract §§4.3, 4.6 |
| D21 | Cleanup finds state by stable API ID and preserves foreign/live resources. Manager tasks own their backend objects; VMaaS owns VM teardown, and core owns dependent-operation ordering and finalizers. | Contract §§4.3, 4.7, 4.10; VMaaS deletion |
| D22 | Netris lab evidence is required for the provisional EVPN segment/gateway duties. Exact Fabric route-type behavior remains open until that test; no fixed type-5 assertion or performance threshold is inferred. | Contract §§4.9, 4.11, 4.13 |
| D23 | Proposed compatibility amendment: every cudn_evpn network emits macVRF.routeTarget = "0:<l2_vni>" and ipVRF.routeTarget = "0:<l3_vni>", matching PR #1971, without a Netris/vendor condition. The selected Fabric must import/export the matching RTs. Shared-contract/Fabric-owner approval and real-pair evidence are required before claiming aligned support. | User direction; osac PR #1971 |
| D24 | Intended later integration consumes the actual RT values exposed by the Fabric/controller API once its supported API and OSAC handoff are agreed. It must use those values rather than keep deriving RTs from VNIs. No new API field name, import/export schema or current availability is invented here. | Earlier user-provided controller integration guidance; Q5 |
| D25 | SecurityGroup enforcement is missing from the current cudn_evpn role and remains mandatory OSAC-6761 implementation: group create/delete, baseline deny, exact ingress/egress rules for VM pods, ComputeInstance-ID binding before boot, updates for every bound target and owned cleanup. Physical-port enforcement remains the Fabric Manager's responsibility whenever BMaaS or CaaS is enabled; Kubernetes policy is not a substitute, and Fabric cannot no-op or report a physical attachment Ready without an effective binding. Q2 resolves the Kubernetes policy mechanism; this work is not deferred or satisfied by the RT fix or a documented no-op. | User reminder; current role task inventory; Contract §4.10 |
| D26 | User-selected proposal: create one VN-owned namespace in create_virtual_network, keep it through attachment/owner-Subnet deletion, and remove it only in safe VN teardown. One NetworkPolicy per group stores VM-pod rules and ComputeInstance-ID membership, with conditional updates preserving the other fields; unbound policy matches no pod. Physical SecurityGroup enforcement remains in the Fabric contract and provider conformance suite. This extends PR #357's manager-specific VN no-op clause. Shared review and Q2/Q3 engineering closure remain required. | User approval of policy-namespace-proposal; 2026-10-25 |

## Earlier Decisions and Their Disposition

The earlier log's identifiers remain traceable here. The baseline copies are retained in the workflow's local revision artifacts; superseded answers are not current installation instructions.

| Earlier ID/reference | Earlier subject | Current disposition |
|---|---|---|
| D1 / R1.Q1 | Administrator performs installation configuration | Retained; D11/D20 define ownership and settings. |
| D2 / R1.Q2 | EVPN details transparent to tenants | Retained; no tenant VNI/RT/BGP configuration. |
| D3 / R1.Q3 | Automatic VNI propagation; no 0:VNI standardization | D12 retains typed VNI propagation; D23 proposes the requested 0:VNI compatibility exception to the D14 baseline. |
| D4 / R1.Q4 | Reject a second Subnet at the API | Superseded by D16: owner-Subnet limit is enforced at overlay attachment, without rejecting sibling Subnets. |
| R1.Q5 | Coordinate two gateway MACs and Fabric-derived DHCP exclusions | Superseded by D13/D15: recorded allocation plan and one Fabric gateway. |
| D5 / R2.Q1 | Manager automatically creates overlay FRRConfiguration | Superseded by D14/D20: installed FRR/VTEP prerequisites and CUDN integration; no per-Subnet peering task. |
| R2.Q2 / D6 / R2.Q3 | VTEP and underlay are prerequisites | Retained and made explicit in the installation assessment. |
| R2.Q4 | Older name/capabilities registration schema | Superseded by D11 and contract §4.2's immutable ansible_role/network_type registration. |
| D7 / R2.Q5 | Obtain route targets from Netris | D23 currently derives explicit compatibility RTs from supplied VNIs. D24 records later authoritative API consumption once the interface is agreed; no such RT field exists in the current typed handoff. |
| D8 / R3.Q1 | Automated real Netris end-to-end validation | Required validation remains; no completed run is asserted. Kind/mocked results are distinct. |
| R3.Q2 | Diagnostics for silent traffic failure | Retained with typed prerequisite/error gates; task success still does not prove packet forwarding. |
| D9 / R3.Q3 | No manager-created MetalLB IPAddressPool | Retained; Fabric/core own the guest-ingress allocation duties. |
| D10 / R3.Q4 | Extend the shared Unified design | Shared requirements remain authoritative; these existing manager documents now specify the manager boundary and delta. |
| Additional subnet-flexibility context | Reuse existing VPC/VNet, different overlay/Fabric subnets | Current target uses one Subnet L2 segment and the recorded allocation pools; no migration/adoption workflow or alternate topology is selected. |

The open [osac PR #1971](https://github.com/osac-project/osac/pull/1971) is included in this proposal through D23. Its code sets both explicit RT fields and adds a render assertion; it does not configure Fabric import/export or implement controller-API RT retrieval. Its conflict with the approved shared contract is resolved only after the matching contract amendment is reviewed, not by describing it as already approved.

## Open Manager Realization Questions

### Q1: Supported DNS and Address-Persistence APIs

- **Owner:** Connectivity & Fabric maintainers, with OVN/KubeVirt integration maintainers.
- **Question:** Which supported hub versions and APIs implement the configured DHCP resolver list and guest address stability through restart/live migration on the primary EVPN network?
- **Impact:** CUDN/attachment implementation and TC-R6-01/TC-ATTACH-02 cannot be finalized from an invented field or assumed platform behavior. These remain required outcomes, not deferred scope.
- **Closure evidence:** Reviewed object/setting examples and a real-hub DHCP/restart/migration result with versions.

### Q2: Policy Objects, Updates and Binding

- **Current status:** The role has no group create/delete or overlay attachment task pair and no policy/binding implementation in its existing Subnet tasks. All D25 outcomes must be implemented and validated under OSAC-6761.
- **Owner:** Connectivity & Fabric maintainers, with OVN policy maintainers.
- **Selected direction:** D26 uses one VN-owned namespace and one NetworkPolicy per group; the user approved proceeding with this proposal. There are no per-attachment copies. The compiler rejects ICMP permanently.
- **Remaining review:** Amend the shared VN/Subnet namespace lifecycle; review the [concrete selector/conditional-update proposal](write-protection-review.md#q2-one-object-policy-updates), capacity and ambiguous-write handling that preserve the required rule/binding guarantees. Q3 must separately establish deletion fencing.
- **Impact:** VN tasks gain namespace side effects; owner Subnet and attachment deletion preserve that namespace and other groups. Single-object updates avoid a copy transaction but do not prove failed-write preservation, delete-wins or DHCP/ARP enforcement.
- **Closure evidence:** Reviewed shared-contract/object/lifecycle examples and API-race/failure tests, followed by deployed enforcement evidence. User selection is not a claim of maintainer approval or implemented lifecycle code.

### Q3: Durable Ownership and Deletion

- **Owner:** Connectivity & Fabric maintainers, coordinated with OSAC-6766 lifecycle owners.
- **Question:** Which durable backend primitive records owner Subnet API ID and detects conflicts without manager Leases, prevents a late create after successful delete, and safely handles a claim while the owner CR is terminating?
- **Investigation result:** The [write-protection assessment](write-protection-review.md#q3-already-issued-creates) finds that a separate durable journal and exclusive worker do not fence an already-issued POST on their own. That candidate remains on hold; unknown creates must keep cleanup pending until a supported mechanism establishes completion. No new private API/worker is selected and no eventual-cleanup exception is adopted.
- **Impact:** Replacing existing Lease/handover logic before establishing these guarantees would weaken cleanup safety.
- **Closure evidence:** Reviewed state transitions, identity predicates and cancelled-create/delete interleavings; no mutable OSAC CR written by the manager.

### Q4: Installation Discovery and Fabric Pair

- **Owner:** Connectivity & Fabric and Enclave installation maintainers, with Netris integration owners.
- **Question:** How does the manager identify the installed VTEP and authoritative hub AS, validate the single-AS prerequisite, and verify the real Fabric's gateway and RT/route behavior?
- **Impact:** Prerequisite checks, AS/VNI rejection, installation examples and supported-pair claims depend on this evidence. No new per-manager tenant API setting is implied.
- **Closure evidence:** Versioned prerequisite inventory and actual lab report. Netris is the initial required real pair; Agentless/Cumulus support requires its own qualifying evidence.

### Q5: Shared Route-Target Amendment and Later API Handoff

- **Owner:** Connectivity & Fabric contract maintainers, with the selected Fabric/controller integration owners.
- **Question:** Approve the all-cudn_evpn explicit 0:VNI exception and matching Fabric import/export duties in contract §§4.9–4.10; determine whether the baseline hub-AS/VNI admission guard changes. When authoritative RTs become available from the controller, agree their API semantics and typed handoff before replacing the compatibility formula.
- **Impact:** A host-only RT change is not universal interoperability: a mismatched Fabric import/export policy can leave BGP up while selected tenant routes are not imported. Current fixtures retain the existing shared VNI guard; the exception alone does not claim a guard change or a ready API.
- **Closure evidence:** Reviewed [shared-contract amendment](route-target-amendment.md) and matching Fabric configuration, exact emitted CUDN values, and real-pair route/traffic evidence. The later API path needs a separately reviewed schema/handoff and tests using API-returned RTs that differ from the VNI-derived formula.

## Tracking and Review

- Documentation: OSAC-6758; manager implementation: OSAC-6761.
- Active shared-core epic: OSAC-6766. Source APIs: OSAC-6768; installation: OSAC-6769.
- OSAC-5974/5976 and OSAC-6753 are superseded tracking, not current implementation dependencies.
- Enclave/plugin/Wizard and console/UX impacts are assessed in the design; conditional drafts are not claimed as created or assigned tasks.
