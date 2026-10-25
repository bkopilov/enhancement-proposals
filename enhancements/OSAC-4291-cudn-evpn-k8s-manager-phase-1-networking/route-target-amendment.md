# Proposed Shared Route-Target Amendment

## Status and Request

Prepared 2026-10-25 in response to the user's continuation after asking whether the manager alignment includes [osac PR #1971](https://github.com/osac-project/osac/pull/1971). This is a concrete review proposal for the shared contract; it is not an approved amendment or a change to the merged PR #357 document.

PR #1971 head `38d18d944035527db02bd95dd903f08b4e9ecedd` adds exactly:

```yaml
macVRF:
  vni: 31
  routeTarget: "0:31"
ipVRF:
  vni: 32
  routeTarget: "0:32"
```

Each RT is derived from its own VNI. Scope is every `cudn_evpn` network; no Netris or vendor condition is added. The PR configures the CUDN and checks rendering; it does not configure the Fabric or retrieve authoritative RTs through an API.

## Required Shared-Contract Edits for Review

Baseline: [PR #357 manager integration contract](https://github.com/osac-project/enhancement-proposals/blob/a430f35da98461b5331f690332b1eebd3a3012b5/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md).

### Section 4.9 — Fabric Route-Target Duty

Replace the `<hub AS>:<VNI>` import/export duty for this compatibility path with:

> When pairing with cudn_evpn using the explicit compatibility RTs, extend the MAC-VRF with matching import/export target 0:<l2_vni> and the IP-VRF with matching import/export target 0:<l3_vni>. Validate the actual matching route policies for the selected pair. A BGP session being established does not by itself establish route import or tenant connectivity.

The reviewed amendment must coordinate the Fabric implementation/settings and installation expectations with the host behavior. It cannot assume that a switch ignores an unmatched RT without affecting connectivity. All other segment/gateway duties retain their existing scope.

### Section 4.10 — Kubernetes Subnet Realization

Replace the `no route targets` phrase in the CUDN creation contract with:

> Set evpn.macVRF.routeTarget to the string 0:<l2_vni> and evpn.ipVRF.routeTarget to the string 0:<l3_vni> for all cudn_evpn networks. Preserve the corresponding VNI and installed VTEP values.

The current `network_segment` remains exactly `{type, l2_vni, l3_vni}`. This compatibility change needs no invented RT input key.

### Existing Hub-AS/VNI Admission Rule

The manager revision retains the baseline guard until the shared maintainers decide its disposition. This proposal does not assert that 0:VNI itself has the hub-AS-derived bound, or silently expand the accepted VNI range. The shared source/API validation, manager checks, Fabric configuration and boundary fixtures must agree on the reviewed outcome.

## Intended Later API Integration

The final integration should consume the actual RT values exposed by the Fabric/controller API, once supported API semantics and the OSAC handoff are agreed. It then uses those authoritative values instead of assuming a VNI-derived formula. Review must define the RT input shape, validation, persistence and cleanup/retry behavior, including whether the API supplies one common RT or separate import/export values; no field name or schema is selected here.

That later change is separate from the two-field compatibility fix and is not claimed implemented or available. Its tests must supply API-returned RTs that differ from 0:VNI and verify that the returned values are used unchanged according to the agreed contract.

## Acceptance and Ownership

- Connectivity & Fabric contract maintainers review the shared amendment and guard disposition with core, installer and Fabric/controller integration owners.
- Manager/render evidence verifies the two exact RTs and all-cudn_evpn scope; existing PR #1971 assertions are retained.
- Real-pair evidence verifies host RTs, both Fabric VRF import/export policies, imported routes, permitted traffic and a controlled mismatch/recovery. Record actual vendor/controller versions; do not infer universal support.
- Updated manager documents, shared contract, Fabric implementation/settings and installation guidance agree before claiming activated contract support.
- Q5 in the revised clarifications tracks this decision. TC-RT-01/02 in the Markdown plan record required coverage; neither is an executed result here.
