# Test Plan — CUDN EVPN Manager Contract Alignment

## Overview

- **Documentation:** OSAC-6758; **manager implementation:** OSAC-6761; **parent:** OSAC-6757 under OSAC-5928.
- **Source:** PR #357 manager contract §§4.3–4.10, Unified/VMaaS designs, this directory's revised PRD/design, and the user-requested PR #1971 RT amendment proposed for shared review.
- **Active cases:** 35. **Retired historical IDs:** 4. Active IC coverage: IC-1 through IC-6.
- **Execution status:** Planned; this document records no executed checks, CI success, packet capture or real-Fabric result.
- **Ownership:** DEV owns Unit, Contract and component integration; QE owns deployed cross-component journeys. Shared-core and installer cases stay in their owning work under OSAC-6766/6769.

The former manager plan's case IDs are preserved where their behavioral subject remains valid. Changed mappings and retired IDs are recorded below. No local decomposition/sync manifest exists for this revision; downstream Jira test references need review when the implementation task is refined.

## Test Boundaries and Planned Invocation

| Tier/owner | Invocation or required runner | Real boundary | Omitted or faked boundary |
|---|---|---|---|
| Unit / DEV | From osac-aap: `uv run pytest tests/unit/test_cudn_evpn_role.py tests/unit/test_cudn_evpn_delete_order.py`, extended during implementation | Input/rule/IPAM transformations, rendering and isolated behavior | Kubernetes, OVN, FRR, KubeVirt and Fabric |
| Contract / DEV | Exact-input/result role fixtures in the component suite; from osac-aap: `uv run pytest tests/unit` after fixtures exist | Real role entry-point invocation where declared; artifact/error serialization | External endpoints are explicit fixtures; no deployed core/AAP or packet enforcement |
| Component integration / DEV | New focused manager target in the existing `tests/integration/targets/` harness, registered in `make test` during OSAC-6761 | Real Ansible processes and Kind APIs; namespace/policy/state writes and conflicts | OVN, FRR, KubeVirt/Fabric packet path and deployed AAP. Installed CRD/prerequisite fixtures are identified as fakes |
| Core/installer integration / owning DEV | Owning existing suites extended under OSAC-6766/6769; applicable AAP `make test` and installer native validation | Declared dispatch/status/installation API boundary | A mocked job cannot prove deployed task ordering or connectivity |
| E2E / QE | Implemented `tests/e2e/` journeys against the deployed hub and real Fabric; actual entrypoint recorded with evidence | Fulfillment/operator/AAP, OVN/FRR/KubeVirt, DHCP/DNS and real Fabric | Nothing replaced silently; enumerate any omitted component per run |
| Documentation review / documentation DEV | Review this four-document set against PR #357 and the implemented examples | Contract/example consistency | No runtime execution |

No dedicated CUDN component target or generic conformance command is claimed to exist yet. OSAC-6766's conformance work must confirm its actual runner. `make test` is a future qualifying invocation only once the manager target is implemented and registered; unrelated passing targets do not satisfy these cases.

## Requirements and Interface Coverage

| Revised PRD scope/outcome | Cases | Design interface |
|---|---|---|
| Compatible manager availability and installation | TC-R1-01, TC-R7-01, TC-SETTINGS-01, TC-PAIR-01 | IC-1 |
| Automatic segment handoff and network realization | TC-R2-01, TC-R2-02, TC-R3-01, TC-INPUT-01, TC-INPUT-02, TC-VNI-01 | IC-2, IC-3 |
| PR #1971 host compatibility and matching Fabric route targets | TC-RT-01, TC-RT-02, TC-R3-01, TC-PAIR-01 | IC-3 |
| Owner Subnet limit while preserving physical siblings | TC-R5-04 through TC-R5-07 | IC-4, IC-6 |
| Disjoint DHCP allocation, gateway and agreed DNS | TC-IPAM-01, TC-R6-01, TC-R4-02, TC-SETTINGS-01 | IC-3 |
| Permitted VM/physical L2 and routed same-VN connectivity | TC-R4-01, TC-R4-02, TC-PAIR-01 | IC-3, IC-6 |
| Baseline and SecurityGroup enforcement/update | TC-POLICY-01 through TC-POLICY-05 | IC-5, IC-6 |
| Physical-port SecurityGroup enforcement | Shared contract TC-SG-02 and TC-BM-01 | Fabric enforcement is required with BMaaS/CaaS; Kubernetes VM policy cannot substitute |
| Stale SecurityGroup status feedback and update serialization | Shared Unified contract TC-SG-UPDATE-STATUS-01 | Status must match the new input before success; another Update stays rejected while Updating |
| Existing VN/Subnet APIs, identities and allocations; fresh-install-only activation | TC-COMPAT-01, TC-COMPAT-02, TC-COMPAT-03; deployed retry in TC-PAIR-01 | IC-1, IC-2, IC-3, IC-4, IC-6 |
| First-boot preparation and stable VM addresses | TC-ATTACH-01, TC-ATTACH-02 | IC-6 |
| Retry, cancellation, owned cleanup and diagnostics | TC-RETRY-01, TC-ATTACH-03, TC-DELETE-01, TC-DELETE-02, TC-R8-01 | IC-2, IC-4, IC-6 |
| Provider backend cleanup after a killed task and late write | Shared contract TC-DELETE-02; CUDN hub-object races TC-R5-07 | Verify ordered cleanup or a fenced late write, with no post-success resurrection |

## Test Cases

### Existing VirtualNetwork and Subnet Compatibility

#### TC-COMPAT-01: Preserve the VirtualNetwork and Subnet API contract

- **Tier / owner:** Core component integration / owning DEV OSAC-6766. **IC:** IC-2, IC-3. **Boundary:** real fulfillment API/persistence and CR projection; manager completion is an explicit fixture, with no Fabric packet path.
- **Preconditions:** Existing PR #357 VN/Subnet client fixtures, tenant defaults and saved IDs/specs/parent references, recorded managers, IPAM and accepted segments.
- **Steps:** Run the unchanged Create/Get/List client fixtures with explicit references and tenant-default resolution. For existing fixtures under the amended lifecycle, compare immutable state before and after namespace/group preparation and reconciliation. Exercise Delete on a separate unreferenced fixture and rejection on a referenced fixture.
- **Expected:** Existing requests need no new namespace/policy field or tenant Update. IDs, names, tenant/parent references, immutable specs, manager identities, CIDRs, gateway, pools and VNIs are retained; internal progress does not allocate again or replace records/CRs. Existing readiness and dependency guards still apply to create/delete.

#### TC-COMPAT-02: Reuse compatible network objects on supported retries

- **Tier / owner:** Component integration / DEV OSAC-6761 with core lifecycle DEV. **IC:** IC-2, IC-3, IC-4, IC-6. **Boundary:** real Kind objects and independent Ansible invocations; OVN/Fabric traffic is omitted.
- **Preconditions:** A compatible recorded VN/owner-Subnet realization from fresh activation of the amended lifecycle, namespace and CUDN, sibling Subnet, group policy and attachment fixtures; snapshot all names, UIDs, owner identities, selectors, accepted segment and IPAM.
- **Steps:** Retry VN/Subnet create and group/binding preparation, inject a response loss/conflict, then restart and reconcile from persisted state.
- **Expected:** Namespace and CUDN retain names and UIDs; VN/Subnet specs, owner, recorded pools/VNIs and attachment namespace remain identical. No second namespace/network, deletion/recreation, reallocation, sibling takeover or VM operation occurs. Policy tasks change only their owned policy/binding state according to selected rules.

#### TC-COMPAT-03: Refuse live legacy activation before mutation

- **Tier / owner:** Core/installer component integration / owning DEV OSAC-6766/6769 with manager DEV. **IC:** IC-1, IC-2, IC-3, IC-4, IC-6. **Boundary:** real activation preflight and Kubernetes inventory; legacy CUDN/VM objects are explicit fixtures and do not prove packet continuity.
- **Preconditions:** Live legacy VN/Subnet records, a Subnet-named namespace, VN-named CUDN, existing selectors/ownership/RTs/IPAM and workload fixtures; capture configuration, API state and object UIDs/specs. Include an RT mismatch and ambiguous legacy ownership.
- **Steps:** Attempt to activate the proposed namespace/policy lifecycle over that inventory.
- **Expected:** Preflight rejects activation with the unsupported legacy-state reason before changing configuration, resetting readiness or dispatching provisioning/cleanup. Records, CRs, UIDs, allocations, selectors, RTs and workload placement remain identical. No `ce-<VN UUID>` replacement namespace, ownership transfer, baseline-deny policy or network deletion is issued. This is rejection coverage, not an in-place migration claim.

### Installation and Typed Handoff

#### TC-R1-01: Resolve the installed compatible manager

- **Tier / owner:** Core/installer component integration / owning DEV. **IC:** IC-1. **Boundary:** real registration/controller API with declared installer rendering; no Fabric traffic.
- **Preconditions:** Installed registration fixtures, networking readiness controller and exact PR #357 registration schema.
- **Steps:** Install the immutable `cudn_evpn` registration with its `ansible_role` and `network_type: evpn`; select a compatible Fabric registration. Repeat with missing role/type, mutable or duplicate registration and incompatible type.
- **Expected:** Valid selection records the OSAC role/type. Invalid selection fails with an actionable readiness diagnostic before manager tasks; it never falls back to another manager. No runtime connectivity claim follows from registration success.

#### TC-R2-01: Pass only the accepted typed segment to Kubernetes

- **Tier / owner:** Core component integration / owning DEV. **IC:** IC-2, IC-3. **Boundary:** controller/job and stored status; Fabric/AAP execution may be fixture-driven.
- **Preconditions:** Subnet stored input/IPAM and controlled Fabric completion fixture.
- **Steps:** Complete Fabric create with different integer L2/L3 VNIs; inspect the accepted segment/status update and subsequent Kubernetes task input. Repeat the Kubernetes task and matching delete.
- **Expected:** Kubernetes create starts only after segment acceptance. It receives `resource`, parent snapshot, recorded `ipam`, exact `network_segment` and explicit tenant; retries/delete reuse those values. No legacy ConfigMap transfer, free-form extraVars or RT field is required.

#### TC-R2-02: Reject invalid or changed Fabric segments before launch

- **Tier / owner:** Core contract/integration / owning DEV. **IC:** IC-2, IC-3. **Boundary:** real typed-output/status acceptance; Fabric output is a fixture.
- **Preconditions:** Controlled job association and Subnet segment fixtures, including another active resource for conflicts.
- **Steps:** Supply missing/extra keys, string/bool VNIs, equal/out-of-range VNIs, wrong type, conflicting VNIs, a foreign job result and a result that changes an accepted segment. Include the documented in-progress-delete conflict wait.
- **Expected:** Malformed/foreign/changed outputs do not replace stored segment or start Kubernetes work. A deleting conflict follows the contract wait/recheck behavior rather than freeing/reusing its VNI early. Tenant diagnostics do not expose other resources' IDs/VNIs.

#### TC-INPUT-01: Exercise all eight entry points and exact outputs

- **Tier / owner:** Contract / DEV OSAC-6761. **IC:** IC-2. **Boundary:** fresh Ansible invocation and captured set_stats; Kubernetes backend is a declared fixture here.
- **Preconditions:** Exact minimal snapshot fixtures for all four task pairs, without tenant annotations or arbitrary full CR fields.
- **Steps:** Invoke each task with one `osac_job_vars`; capture artifacts and process outcome. Run proposed VN namespace create/delete and non-owner Subnet no-ops as well as overlay apply.
- **Expected:** All required files exist. No-output success publishes no `osac_result`; apply publishes exactly `{vm_attachment: {namespace}}`. No version/profile/identity wrapper or secret enters artifacts. State changes use top-level API IDs and explicit tenant. Proposed VN tasks create/safely remove their shared namespace without typed output; they are no longer documented no-ops.

#### TC-INPUT-02: Reject invalid identities and sanitize failures

- **Tier / owner:** Unit/Contract / DEV OSAC-6761. **IC:** IC-2. **Boundary:** input checks and real error serialization; failing backend is a fixture.
- **Preconditions:** Valid input controls and a write-recording backend fixture containing deliberately sensitive error arguments.
- **Steps:** Vary missing/mismatched IDs, tenant/parents, manager/task, resource kind, target/interface and duplicate group IDs. Inject a backend timeout and a permanent unsupported rule.
- **Expected:** Invalid inputs fail before writes. Reported errors contain only nonempty message and boolean retryable, with the expected classification; the play fails. No raw failed-result/invocation, token or secret value is emitted.

#### TC-SETTINGS-01: Make missing prerequisites recoverable

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-1, IC-2, IC-3. **Boundary:** real Kind objects and Ansible environment lookup; prerequisite objects are schema fixtures, not live FRR/OVN.
- **Preconditions:** Owner-capable Subnet input and a reviewed prerequisite discovery mechanism from Q4.
- **Steps:** Omit or malform DNS resolver setting, then omit installed VTEP/required FRR/hub prerequisite. Repair one prerequisite at a time and repeat the same input.
- **Expected:** Each missing/malformed setting/prerequisite fails retryably and names the item without a success artifact. Repair permits convergence without duplicate state. No task silently installs peering or substitutes a default resolver.

### CUDN and Address Allocation

#### TC-R3-01: Realize the exact primary EVPN network

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-3. **Boundary:** real Kind API plus declared CUDN/prerequisite schema fixtures; no OVN dataplane.
- **Preconditions:** Valid owner input, exact recorded IPAM/segment and configured prerequisites.
- **Steps:** Run create twice; inspect persisted CUDN, owner/tenant metadata and selected namespace identity. Compare with recorded input rather than recomputed ranges.
- **Expected:** One primary Layer2 CUDN has EVPN transport, installed VTEP, supplied L2/L3 VNIs, MTU 1500, full Subnet CIDR, recorded gateway and every non-overlay exclusion. Under the proposed PR #1971 amendment, macVRF.routeTarget is "0:<l2_vni>" and ipVRF.routeTarget is "0:<l3_vni>". No overlay gateway/router-port resource or manager-created peering configuration appears. Actual OVN readiness and matching Fabric RTs are covered by deployed cases; shared-contract approval is a gate.

#### TC-RT-01: Preserve PR #1971's two explicit route targets

- **Tier / owner:** Unit/Contract / DEV OSAC-6761. **IC:** IC-3. **Boundary:** real Jinja/role rendering; no switch or BGP session.
- **Preconditions:** PR #1971's existing render assertion and valid distinct typed segment fixtures representing different Fabric producers; the role input contains no vendor selector.
- **Steps:** Render MAC VNI 31 and IP VNI 32, then repeat with another valid pair and producer-labelled fixtures that supply the same prescribed input shape. Inspect the exact EVPN structure and repeat the render/task input.
- **Expected:** Values are strings "0:31" and "0:32" in the matching MAC/IP fields; each always follows its respective VNI. The rule applies whenever cudn_evpn is selected, without a Netris-only condition. Other EVPN values and deterministic retry behavior remain correct. Rendering proves only emitted host configuration; it does not establish Fabric acceptance.

#### TC-RT-02: Verify matching Fabric import/export and expose mismatch

- **Tier / owner:** E2E / QE with Fabric integration owners. **IC:** IC-3. **Boundary:** real OVN/FRR and the selected Fabric's MAC/IP VRFs.
- **Preconditions:** Reviewed Q5 amendment and an authorized lab with the explicit host RTs and a matching configured Fabric pair; packet probes otherwise permitted by policy.
- **Steps:** Observe the CUDN RTs and Fabric MAC/IP import/export configuration, imported routes and successful permitted traffic. In the lab, introduce a controlled RT mismatch, observe the affected routes/traffic, then restore the matching configuration. Record each claimed vendor/controller/version separately.
- **Expected:** The matching pair exchanges the required tenant routes and forwards permitted traffic. The mismatch is detected through RT/route and traffic evidence even if BGP remains established. Restoring matching policies restores the path. No Arista/Cumulus/FRR default or universal pair support is inferred; a host-only change cannot satisfy this case.

#### TC-VNI-01: Enforce the hub-AS/VNI representation limit

- **Tier / owner:** Unit/Contract / DEV OSAC-6761. **IC:** IC-2, IC-3. **Boundary:** validation and recorded side effects; hub-AS source is a fixture.
- **Preconditions:** Valid segment/control input and authoritative AS values from the reviewed discovery path.
- **Steps:** Exercise hub AS 65535 and 65536 with VNIs 65535/65536; include VNI 1/16777215 boundaries, 0, above maximum, equal IDs, bool and numeric strings.
- **Expected:** Contract-valid integers pass the applicable AS constraint. With AS above 65535, any VNI above 65535 fails permanently before mutation with a message naming the rule. Invalid scalar types/ranges/equality never create a network. This case retains PR #357's current shared admission rule pending Q5; it does not claim the bound is intrinsic to explicit 0:VNI or that PR #1971 implements a new guard.

#### TC-IPAM-01: Preserve the recorded allocation plan

- **Tier / owner:** Unit / DEV OSAC-6761. **IC:** IC-3. **Boundary:** IPAM validation/exclusion transform and render; no DHCP server.
- **Preconditions:** Recorded layouts for split and non-power-of-two shares, with empty physical/lbVip lists and disjoint valid overlay ranges.
- **Steps:** Produce the CUDN exclusions; enumerate their union against the full Subnet minus overlay ranges. Vary canonical form, overlap/out-of-Subnet input and missing/incorrect gateway.
- **Expected:** Exclusions exactly cover all non-overlay addresses, including guest-ingress pools, without removing usable overlay addresses. No competing plan or Fabric-derived gateway is calculated. Malformed layouts fail before writes; network/broadcast/gateway are never offered as workload allocations. Empty lists remain valid where the contract permits them.

#### TC-R6-01: Observe real DHCP pools, Fabric gateway and DNS

- **Tier / owner:** E2E / QE. **IC:** IC-3, IC-6. **Boundary:** deployed OVN DHCP, KubeVirt guest, real Fabric/physical DHCP and resolvers.
- **Preconditions:** Supported reviewed Q1/Q4 mechanisms and an owner Subnet with all nonempty pools, VM and physical client.
- **Steps:** Boot clients, capture leases and query agreed DNS; renew leases and sample multiple addresses. Inspect gateway ARP and DHCP server behavior with restrictive application policy.
- **Expected:** VM addresses belong only to overlay pools; physical addresses only to Fabric pools; neither consumes VIP/network/broadcast/gateway addresses. Guests get the full Subnet prefix, recorded Fabric gateway and exact configured resolvers. DNS resolution succeeds. Servers answer only their own interfaces; gateway ARP and DHCP remain usable under baseline restrictions.

### Owner Subnet and Deletion Races

#### TC-R5-04: Let the first executing create claim ownership

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-4. **Boundary:** real Kubernetes state and independent Ansible runs, with core ordering represented explicitly.
- **Preconditions:** Two sibling Subnet snapshots with reversed creation-time versus execution order; unowned VN state.
- **Steps:** Run the newer Subnet's create first, then the older one, and retry both. Exercise the default-Subnet ordering as a shared-core fixture.
- **Expected:** The first successful claim records that Subnet's API ID, independent of timestamp/name. Exactly one network exists. Retries do not change ownership. Default ordering is a core guarantee, not a manager timestamp election.

#### TC-R5-05: Preserve owner VMs when another Subnet is added

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-4, IC-6. **Boundary:** real Kind network/state/policy objects; VM execution is omitted.
- **Preconditions:** Owned network and owner binding plus a sibling's valid recorded plan/segment.
- **Steps:** Run sibling create/delete and an overlay apply/delete for it; inspect owner resources and recorded sibling plan.
- **Expected:** Sibling Subnet tasks succeed without creating a second network; overlay apply fails permanently naming the owner-network limitation and writes no attachment artifacts. Its absent delete succeeds. Owner bindings/network and both recorded plans remain unchanged. A core/Fabric companion check establishes that physical sibling attachments remain available.

#### TC-R5-06: Release ownership without handover

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-4. **Boundary:** real Kind objects and Ansible cleanup.
- **Preconditions:** Owner Subnet, a surviving sibling and dependency-ordered attachment cleanup already complete.
- **Steps:** Delete the owner, repeat delete, then inspect the sibling; run a subsequent new Subnet create. Include no-op deletion of a non-owner and already-absent state.
- **Expected:** Owner network/baseline are removed before claim release; the VN-owned namespace and surviving group policies remain. A later owner reuses that namespace without inheriting old bindings. No surviving sibling is promoted. A subsequent create can claim the released network. Non-owner/absent cleanup does not delete foreign or newly claimed state.

#### TC-R5-07: Exercise owner termination and late create/delete writes

- **Tier / owner:** Component integration / DEV OSAC-6761 with core lifecycle DEV. **IC:** IC-4. **Boundary:** real API conflicts and independently paused Ansible/backend operations for Kubernetes Manager-owned hub objects; no packet path. The provider Fabric backend is covered separately by shared contract [TC-DELETE-02](/enhancements/OSAC-1433-network-manager-integration-contract-networking/testplan.md#tc-delete-02).
- **Preconditions:** Reviewed Q3 ownership/delete-wins mechanism; controls that pause/release exact Namespace, CUDN and NetworkPolicy mutations, expose cleanup progress/waiting and authoritative settlement, and record commit/rejection order. Keep a separate live ownership control and snapshot its state. Use the configured cleanup deadline as a failure bound, not proof of mutation completion.
- **Steps:**
  1. Start another create while the owner CR is terminating; finish owner cleanup and retry the claim.
  2. Separately pause an already-issued create mutation before commit, cancel its job and start matching deletion with the stored input.
  3. Observe deletion reaching cleanup, waiting for the outstanding operation, or succeeding; then release the exact mutation without waiting for deletion to succeed first.
  4. Await operation settlement and successful cleanup within the configured deadline. Repeat the trace for Namespace, CUDN and NetworkPolicy.
  5. Inspect owned objects and mutation history before a repeated delete or corrective cleanup. Verify the live control is unchanged, then require repeated deletion to succeed. Separately race cleanup from an old API ID with a new claim and verify the new owner's objects survive.
- **Expected:** A terminating owner is treated as present and produces a retryable claim failure; retry after release can claim safely. Ordered cleanup may stay pending until the held operation settles, then removes all state before success. If delete succeeds first, the released write must be rejected or become a no-op through an enforced backend fence; no later or transient recreation is permitted. Post-success corrective cleanup cannot hide a leak. Old cleanup never removes the new owner's objects. A preflight tombstone read or journal alone does not prove safety; unknown/indefinitely pending operations do not pass. No manager Lease is required. The shared [TC-DELETE-02 oracle](/enhancements/OSAC-1433-network-manager-integration-contract-networking/testplan.md#tc-delete-02) defines safe ordered/fenced and unsafe fixture controls.

### Policy and Namespace Preparation

Current implementation gap: the manager's SecurityGroup and policy-binding work has not been implemented. TC-POLICY-01–05 and the attachment cases below are required OSAC-6761 coverage. Their presence in this plan is not passing evidence, and the RT fix does not satisfy them.

#### TC-POLICY-01: Create unbound groups and update the shared policy

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-5. **Boundary:** real policy/state API objects; no packet enforcement.
- **Preconditions:** Reviewed D26/Q2 shared VN namespace model, VN snapshot and group rules; initially no Subnet, network or attachment.
- **Steps:** Run VN create twice, then create the group before a Subnet exists. Bind two target IDs in the shared namespace, replace rules with a rerun, retry, then delete after unbinding and repeat deletion. Include another group's policies and a foreign-tenant ownership control.
- **Expected:** Initial group creation needs no network and selects no VM. One VN-owned namespace and one SG-owned policy exist; all bound target IDs follow its accepted replacement. Retries create no duplicates. Group deletion removes only its policy and preserves the namespace. Repeated deletion succeeds; other groups and tenant state remain unchanged. No attachment rerun is needed for the update.

#### TC-POLICY-02: Reject unsupported policy and preserve empty directions

- **Tier / owner:** Unit/Contract / DEV OSAC-6761. **IC:** IC-2, IC-5. **Boundary:** rule translation and captured object replacements; Kubernetes policy engine omitted.
- **Preconditions:** Last-successful group rules and exact TCP/UDP/all-protocol inputs, empty directions and ICMP cases.
- **Steps:** Render both policy directions, port boundaries/ranges and remote CIDRs. Attempt malformed/unsupported rules and inject replacement failure. Exercise the selected ICMP implementation or permanent rejection path.
- **Expected:** Every NetworkPolicy includes Ingress and Egress, even with no rules in one direction. Rules retain exact allowed traffic semantics. Unsupported ICMP is never silently dropped; rejection is permanent before mutation. A failed replacement retains the object's previous rules.

#### TC-POLICY-03: Deny unbound traffic while preserving DHCP/ARP

- **Tier / owner:** E2E / QE. **IC:** IC-5. **Boundary:** actual primary EVPN dataplane.
- **Preconditions:** Ready owner network, baseline installed, unbound VM/import or other pods and a reachable test endpoint.
- **Steps:** Probe application ingress/egress from unbound pods; perform DHCP renewal and gateway/peer ARP. Where backend supports the restriction, send a tenant-originated DHCP reply.
- **Expected:** Application traffic is blocked both ways before any binding. Owning DHCP and permitted ARP succeed. Supported rogue-DHCP protection blocks tenant replies. Results identify actual backend/version and any unimplemented protection explicitly.

#### TC-POLICY-04: Enforce selected group union and updates

- **Tier / owner:** E2E / QE. **IC:** IC-5, IC-6. **Boundary:** real VM/Fabric traffic and policy engine.
- **Preconditions:** Two groups with distinct ingress/egress allows, multiple bound VMs and an unbound control pod; known last-successful rules.
- **Steps:** Probe allowed/denied TCP/UDP/all cases and stateful replies, replace a group's rules, probe each bound target in the shared namespace, and exercise the permanent unsupported-ICMP path.
- **Expected:** Each VM receives exactly its selected union in both directions, follows the successful update without reattachment, and permits stateful replies. Empty directions allow no application traffic through that policy contribution. Unbound/control traffic remains protected. Unsupported ICMP is a visible failed group, not an unfiltered success. This case proves VM-pod enforcement only; physical-port enforcement is separately required by shared contract TC-BM-01, and a Fabric no-op is nonconformant whenever BMaaS or CaaS is enabled.

#### TC-POLICY-05: Converge a bind racing a rule replacement

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-5, IC-6. **Boundary:** real policy/state API conflicts with controlled independent Ansible operations.
- **Preconditions:** Reviewed Q2 atomicity/conflict algorithm and a group with two rule generations.
- **Steps:** Pause apply after reading group state, replace rules, resume bind; reverse the order, inject resourceVersion conflicts/failed writes and attempt an older rule generation. Rerun the permitted retry paths.
- **Expected:** A successful bind/update converges to the accepted current rule state; conflict is retryable rather than a stale success. Failed replacement preserves last-successful rules according to the reviewed backend guarantees. No manager lock or VM inspection is used.

#### TC-ATTACH-01: Prepare deterministic namespace and policy before a VM

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-2, IC-5, IC-6. **Boundary:** real Kind namespace/selection/policy writes and captured result; no VM CR is installed.
- **Preconditions:** VN-created namespace, owner network and existing group policies; two target ComputeInstance API IDs; absence of VM/VMI APIs proves no dependency on VM reads.
- **Steps:** Apply both targets twice and across fresh Ansible processes. Inspect shared namespace labels, primary-network selection, policy membership selectors and set_stats. Supply a foreign-tenant namespace collision and mismatched owner input.
- **Expected:** Both targets return the same VN-owned namespace on every retry, with the exact tenant label, matching primary network and membership selectors on `osac.openshift.io/compute-instance-id`. Only namespace is returned. Collisions fail without editing foreign state. Core consumer fixtures show VM creation waits for accepted preparation and supplies its own labels/l2bridge.

#### TC-ATTACH-02: Keep the VM address through restart and migration

- **Tier / owner:** E2E / QE. **IC:** IC-3, IC-6. **Boundary:** actual KubeVirt restart/live migration, OVN allocation and Fabric binding.
- **Preconditions:** Reviewed Q1 supported mechanism, migratable VM, observed overlay address and completed Fabric policy bind.
- **Steps:** Record address/DHCP/network state, restart the VM, then migrate it to another eligible hub node. Observe address and permitted traffic after each transition without a new allocation/bind input.
- **Expected:** The same address is retained within the recorded overlay pool; Fabric traffic continues with the existing binding after convergence. No per-operation RT or duplicate allocation is introduced. Record actual supported versions and interruption measurements without inventing a latency SLA.

#### TC-ATTACH-03: Clean only owned attachment resources

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-4, IC-6. **Boundary:** real namespace/policy/state API objects.
- **Preconditions:** Two live target bindings and a foreign-owned control in the VN-owned namespace; groups contain current rules.
- **Steps:** Delete one attachment twice; exercise partial apply state and an unsupported Subnet attachment that created nothing. Attempt cleanup with stale/foreign ownership and with another live resource present.
- **Expected:** Only the target's owned binding/artifacts disappear; the other binding, shared namespace and foreign objects remain. The VN namespace is never removed by attachment deletion; other memberships and current group rules remain unchanged. Never-created cleanup succeeds. No VM/VMI is read, patched or deleted by the manager.

### Retry, Deletion and Operational Evidence

#### TC-RETRY-01: Resume partial work and clean after cancellation

- **Tier / owner:** Component integration / DEV OSAC-6761. **IC:** IC-2, IC-4, IC-5, IC-6. **Boundary:** fresh Ansible processes and real API state, with controlled failure injection.
- **Preconditions:** Fixtures for all eight task entry points and barriers after each meaningful backend write.
- **Steps:** Interrupt creates/applies at each write; retry with the same stored input or invoke matching delete. Include process loss before request submission, after submission, and after backend commit before acknowledgement; transient API failure; permanent unsupported input; and a delete before create ever succeeded.
- **Expected:** Retries converge with the same identities/output, without duplicates or changed addresses/owners. Unsupported input is final, transient faults publish retryable errors, and deletes complete owned cleanup from partial/absent state. An unknown create outcome keeps cleanup retryably pending; an empty GET or timeout alone cannot produce a successful delete. Indefinite unknown state does not establish restart recovery. Failed deletes remain failures for core to retry; no success artifact conceals a cleanup error.

#### TC-DELETE-01: Observe VMaaS-owned teardown before network cleanup

- **Tier / owner:** E2E / QE with core lifecycle DEV fixtures. **IC:** IC-4, IC-6. **Boundary:** deployed VMaaS/operator/AAP/OVN/Fabric.
- **Preconditions:** Running owner VM and selected group; sibling/foreign live controls; ability to inject a manager cleanup failure.
- **Steps:** Delete the VM; observe VMaaS removal before Fabric then Kubernetes attachment deletes. Delete the owner Subnet after attachments, then its groups/VN through dependency guards. Retry a failed cleanup.
- **Expected:** Manager tasks never delete the VM. Finalizers retain ownership until cleanup succeeds. Network/baseline removal follows attachment teardown and releases the owner without handover. The VN namespace and surviving groups remain through Subnet deletion; final VN cleanup removes its namespace only after child teardown and foreign/live-resource safety checks. Foreign/sibling live resources remain, and no released VNI/state is reused before its owning Fabric delete succeeds.

#### TC-DELETE-02: Review stale-VRF support without automatic disruption

- **Tier / owner:** Documentation review / documentation DEV; any recovery execution needs the platform team's approved lab procedure. **IC:** IC-4.
- **Preconditions:** A deleted CUDN and evidence of an apparent node VRF remnant; inventory of remaining live networks/attachments.
- **Steps:** Verify the support guidance requests ownership/OVN/FRR evidence and platform review; inspect manager delete behavior for disruptive host operations.
- **Expected:** Guidance distinguishes a remnant from a live owned network. Routine delete never restarts networking pods or deletes host VRFs. Any platform recovery records its effect on other node workloads; no direct-delete recipe is treated as universally safe.

#### TC-R7-01: Review installation and Enclave handoff guidance

- **Tier / owner:** Documentation review / documentation DEV, coordinated with OSAC-6769 and Enclave owners. **IC:** IC-1.
- **Preconditions:** Current installation reference, confirmed plugin/Wizard consumer/version when available.
- **Steps:** Review VTEP/FRR/node peering, shared hub AS/VNI/MTU, underlay VTEP routing, DNS setting/environment and immutable registration. Trace installer schema → plugin → Wizard controls/errors.
- **Expected:** Requirements and diagnostic examples distinguish the approved baseline from the proposed RT and VN namespace lifecycle amendments; Fabric owns the gateway, and the compatibility example shows both explicit 0:VNI host fields plus required matching Fabric import/export. Required consumer/amendment work is tracked with clear ownership/status; proposed drafts are not described as approved contracts, created assignments or verified runtime support.

#### TC-R8-01: Review actionable diagnostic and ownership guidance

- **Tier / owner:** Documentation review / documentation DEV. **IC:** IC-1, IC-2, IC-4.
- **Preconditions:** Sanitized example task errors and CUDN/namespace/FRR/Fabric evidence.
- **Steps:** Follow guidance for missing prerequisites, AS/VNI rejection, owner-limit failure, policy rejection, transient conflict and traffic failure despite successful tasks.
- **Expected:** Each path identifies the responsible setting/owner and supported repair/lifecycle action. Read-only commands show segment/route/peering/ownership evidence. No secret, other-tenant IDs, arbitrary finalizer removal or unsupported adoption path is exposed.

#### TC-PAIR-01: Record the supported real EVPN pair

- **Tier / owner:** E2E / QE with Fabric and OVN integration owners. **IC:** IC-1, IC-3, IC-5, IC-6. **Boundary:** complete deployed path and real Netris initially.
- **Preconditions:** Q1–Q5 closure, approved shared RT amendment, qualifying hub/Fabric infrastructure, applicable service profiles and actual common-runner entrypoint when supplied by core.
- **Steps:** Execute applicable VM/task profiles and the referenced traffic/lifecycle cases. With workloads running under the supported amended contract, repeat reconciliation and manager retry/restart; compare VN/Subnet IDs/specs, stored pools/VNIs, namespace/CUDN UIDs, guest addresses and allowed/denied probes before and after. Capture actual versions, inputs, gateway/RT/route behavior, commands and each pass/fail/skip. Repeat separately for any additional claimed Fabric pair.
- **Expected:** Report distinguishes real endpoints from fixtures and required skips from passes. It resolves the provisional Fabric route behavior from evidence. Supported retry/restart retains resource and backend identities, pools/VNIs, VM placement/addresses and the same allowed/denied probe outcomes without network or workload recreation. Registration/unit/Kind success is not accepted as proof of an Arista/Cumulus/Netris packet path.

#### TC-R4-01: Prove permitted same-Subnet VM-to-physical L2 traffic

- **Tier / owner:** E2E / QE. **IC:** IC-3, IC-5, IC-6. **Boundary:** real Fabric, VM and physical client.
- **Preconditions:** VM on owner Subnet and physical peer on that same L2 segment, completed bindings and policy permitting a known TCP/UDP probe.
- **Steps:** Probe both directions with allowed policy, remove the necessary allow at one enforcement point, and repeat; inspect actual MAC/IP and EVPN learning.
- **Expected:** Allowed probes and replies succeed, one-sided policy denial blocks application traffic, and the learned path belongs to the recorded segment/VTEP. No unapproved latency threshold or unsupported ICMP assumption determines success.

#### TC-R4-02: Prove Fabric-routed traffic within one VirtualNetwork

- **Tier / owner:** E2E / QE. **IC:** IC-3, IC-5, IC-6. **Boundary:** real Fabric IP-VRF, owner VM and physical peer on a sibling Subnet.
- **Preconditions:** Same VN/L3 VNI, distinct Subnet/L2 VNIs, a valid owner overlay attachment and a physical sibling attachment, with permitted policy.
- **Steps:** Probe the remote Subnet through the recorded Fabric gateway and inspect ARP, routes and the actual EVPN path. Repeat with policy denial and an unrelated overlapping-CIDR VN as an isolation control.
- **Expected:** Permitted same-VN routed traffic succeeds; denied/unrelated-VN traffic fails. The Fabric supplies routing without a second CUDN gateway or inter-VRF-peering assumption. Record actual supported route types; do not require type-5 until the lab establishes that behavior.

## Historical Mapping Changes and Retired IDs

| Existing ID | Earlier requirement/IC mapping | Current disposition |
|---|---|---|
| TC-R1-01 | R1 / IC-1 | Retained; exact immutable registration now belongs to core/installer integration |
| TC-R2-01, TC-R2-02 | R2 / IC-3 | Retained; now IC-2/IC-3, typed accepted segment replaces legacy extraVars extraction |
| TC-R3-01 | R3 / IC-5 | Retained; now IC-3, exact realization and real API boundary |
| TC-R4-01, TC-R4-02 | R4 / IC-5/IC-6 | Retained; now IC-3/IC-5/IC-6, policy-controlled probes and observed route types |
| TC-R5-01 | R5 / IC-2 | Retired: second-Subnet API rejection is superseded by owner-limit cases TC-R5-04–07 |
| TC-R5-02 | R5 / IC-2 | Retired: another Kubernetes Manager type is not this release's supported path |
| TC-R5-03 | R5 / IC-2 | Retired: skip annotation is not the approved owner-limit interface; TC-R5-05 covers allowed physical siblings |
| TC-R6-01 | R6 / IC-5 | Retained; now IC-3/IC-6, recorded plan and real DHCP/DNS replace inferred Fabric exclusions |
| TC-R7-01, TC-R8-01 | R7/R8 / no IC | Retained; now IC-1 and IC-1/IC-2/IC-4 respectively |
| TC-R9-01 | R9 / no IC | Retired: dual gateway MAC coordination is replaced by the Fabric-only gateway contract; covered by TC-R6-01/TC-R4-02 |
| TC-DELETE-01 | Deletion / IC-5 | Retained; now IC-4/IC-6, VMaaS owns VM removal and core orders cleanup |
| TC-DELETE-02 | Recovery / no IC | Retained as documentation review, IC-4; no unverified destructive platform recovery procedure |

Earlier R1–R9 labels were derived test-plan groupings, not a numbered requirements section of the OSAC PRD template. This revision maps cases to named current PRD outcomes instead of adding invented FR IDs.

## Gaps and Evidence Still Required

1. Q1–Q4 engineering closure, D26's shared VN namespace lifecycle amendment and Q5's RT amendment/API handoff remain review gates. Their cases are required coverage, not implementation or lab passes. The later API path needs an agreed schema and separate fixtures using authoritative RT values that differ from the compatibility formula.
2. The manager-focused Ansible/Kind integration target and all newly required task/policy/attachment coverage must be implemented under OSAC-6761. Existing render/task-structure checks are insufficient for real API boundaries.
3. Shared snapshots, saved IPAM, output validation and VM/installer activation remain OSAC-6766/6768/6769 work. Their cases cannot be satisfied by a standalone manager fixture.
4. DNS/policy/address-stability evidence requires actual OVN/KubeVirt; claim/delete races require real API concurrency. Fakes and parsing do not establish those guarantees.
5. Real Netris EVPN interoperability and the common conformance invocation remain unexecuted/unconfirmed here. Additional Fabric pairs, including Arista or Cumulus, require separate reports.
6. Preservation and legacy-activation refusal in TC-COMPAT-01–03 require core/installer/manager implementation and executed evidence. They retain the fresh-install-only boundary and do not establish legacy adoption support. Deployed retry/address/traffic continuity remains TC-PAIR-01 evidence.
7. No Jira issue, assignment or completed conformance result is created by this plan. Refresh downstream task/test references after documentation review.
