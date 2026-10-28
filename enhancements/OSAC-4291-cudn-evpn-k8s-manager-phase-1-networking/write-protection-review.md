# CUDN EVPN Write Protection: Shared-Core Review

**Status:** Open design questions for OSAC-6758/OSAC-6761, coordinated with [OSAC-6766](https://redhat.atlassian.net/browse/OSAC-6766). This note accompanies the [manager design](design.md). It proposes no approved private API, database schema, worker, permission change or weakening of cleanup guarantees.

## Required Behavior

The approved [manager contract §4.7](https://github.com/osac-project/enhancement-proposals/blob/a430f35da98461b5331f690332b1eebd3a3012b5/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md) allows core to start deletion after the cancelled AAP job ends, without waiting for its backend work to finish. Delete succeeds only when cleanup is complete. A late write for the deleted API ID must leave no state after that success. Enforced backend tombstones or ordered requests are expressly allowed.

Partial state while deletion is pending is not forbidden by this requirement. Core must retain the finalizer when the manager cannot establish completion. AAP cancellation, an ended job and an absent-object read are not individually backend completion acknowledgements.

The SG task also requires atomic rule replacement, preservation of previous rules on failure, and removal of policy objects on deletion. The questions below preserve these requirements pending shared review.

## Q2: One-Object Policy Updates

The selected namespace proposal uses one NetworkPolicy per group. Rules and target membership share its identity but use separate conditional patches:

| Mutation | Preconditions | Changed fields | Preserved fields |
|---|---|---|---|
| Rule replacement | Verify tenant/owner, UID, resourceVersion, source generation and rule digest | ingress, egress, policyTypes, rule generation/digest annotations | Target selector and unrelated metadata |
| Bind/unbind | Verify tenant/owner, UID, resourceVersion and canonical membership | Selector membership | Rules, generation/digest and other targets |

An unbound selector combines `Exists` and `DoesNotExist` on `osac.openshift.io/compute-instance-id`. A bound selector has one `In` expression with sorted, unique target API IDs. The contradictory unbound requirements follow Kubernetes AND semantics and the inspected validator; installed-cluster admission and selection remain untested. Never use an empty selector for an unbound group. [Kubernetes selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/), [pinned validator](https://github.com/kubernetes/apimachinery/blob/v0.34.1/pkg/apis/meta/v1/validation/validation.go).

Each patch tests UID and the opaque resourceVersion before replacing its fields. Conflicts are retryable; do not force apply or delete/recreate as a conflict workaround. Older rule generations cannot overwrite newer rules. Equal generation with a different compiled digest is a conflict. These are design predicates requiring implementation and real-API validation. [Kubernetes conditional updates](https://kubernetes.io/docs/reference/using-api/api-concepts/#updates-to-existing-resources).

### Ambiguous replacement result

A response can be lost after an atomic replacement commits. Proposed clarification for contract-owner review:

> Rule replacement is atomic. A definitively rejected replacement leaves the prior rules. A transport failure may have an unknown outcome; the manager reports a retryable error and reconciles the recorded desired input without claiming rollback. Newer rules and deletion remain protected from stale attempts.

This wording is not adopted by this PR. Reviewers must reconcile it with the existing failure-preservation clause before implementation acceptance. An unconfirmed timeout must not be reported as proof that the previous rules remain.

## Q3: Already-Issued Creates

The failure trace applies separately to VN namespaces, cluster-scoped CUDNs and group policies:

1. An old create prepares a POST and pauses after its final ownership/tombstone read.
2. Delete records closure, removes owned objects, observes absence and reports success.
3. The old POST resumes and creates an object for the deleted ID.

Kubernetes permits reuse of a deleted object's name with a different UID. UID/RV preconditions on the old object do not prohibit a new POST. A separate ConfigMap or database tombstone checked before the POST has the same gap unless the actual mutation atomically enforces it. This is reasoning from the API model, not an executed race result. [Object names and UIDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/).

### Evaluated coordinator: insufficient on its own

A candidate used the existing fulfillment PostgreSQL deployment for durable attempt/closure records, a private coordinator API and a worker as the sole hub writer. It could close new admission and preserve unknown operations. It does not, by itself, fence a Kubernetes request the worker already sent.

| Source finding | Consequence |
|---|---|
| The inspected timeout handler can respond before its request-handler goroutine finishes. | Timeout is not a handler-completion barrier. |
| etcd documents uncertainty after timeout or a network disruption. | A timeout alone cannot establish definitive non-commit. |
| Kubernetes create invokes a conditional put for the object's own key with expected revision zero. The inspected etcd helper compares that same key. | The create transaction does not test the proposed separate closure record. |

Sources: [Kubernetes v1.34.1 timeout filter](https://github.com/kubernetes/kubernetes/blob/v1.34.1/staging/src/k8s.io/apiserver/pkg/server/filters/timeout.go#L76-L137), [etcd completion guarantees](https://etcd.io/docs/v3.6/learning/api_guarantees/#operation-completed), [Kubernetes create storage path](https://github.com/kubernetes/kubernetes/blob/v1.34.1/staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go#L238-L300), [etcd conditional put](https://github.com/etcd-io/etcd/blob/v3.6.4/client/v3/kubernetes/client.go#L77-L99). These source versions establish the inspected primitives, not a verified installed OCP support matrix.

The replacement worker can keep deletion pending when a create is unknown. However, persisting Started and crashing before the request is sent leaves the same durable record as losing the response after submission. An absent GET does not distinguish those cases. The candidate therefore lacks demonstrated automatic recovery for that ambiguity and remains on hold as an implementation architecture. This is a bounded finding, not a claim that every Kubernetes-based solution is impossible.

### Narrower argument for conditional binding patches

An exclusively controlled writer could persist an exact UID/RV-guarded patch before atomically marking it Started against open scopes. After closure blocks new admission, cleanup removes the target and forces an actual policy write that changes resourceVersion, even if the target was already absent. An older fixed patch either preceded cleanup or fails its old preconditions afterward.

This reasoning assumes every affected policy is durably known, no direct legacy writer exists, and rereading/rebasing the request requires fresh authorization. It does not establish a fence for POST/upsert or remove the need to handle SG/VN deletion and process restarts. It is proposed reasoning for a real-API case, not implemented or passing evidence.

## Decisions Requested From Shared-Core and Contract Owners

1. Name the backend primitive that enforces closure against already-issued creates for all three object kinds. Specify its atomic boundary, restart behavior, ownership checks and retention. A preflight read, worker lease or arbitrary wait does not supply that proof.
2. If retained inert objects are proposed as enforced tombstones, reconcile their lifecycle with the explicit policy-object removal clause, storage/capacity limits and VN/CUDN cleanup. Such a change needs explicit review. Eventual removal would also change the current post-success guarantee and is not selected here.
3. Resolve ambiguous rule-update responses without promising rollback after an unknown outcome.
4. Preserve stable API-ID ownership, tenant/owner metadata, no manager application locks, no VM/VMI access and no deletion of foreign/new incarnations.

Do not implement the candidate's new API/worker solely to move the unresolved create race into another process. The manager's full SG lifecycle remains required; the route-target patch and pure compiler do not provide persistence, baseline enforcement, binding or cleanup.

## Required Evidence

The [test plan](testplan.md) retains TC-R5-07, TC-POLICY-05 and TC-RETRY-01. For create/delete overlap, pause an issued mutation, start cleanup and observe its progress or waiting state, then release the mutation without requiring delete success first. An ordered backend may keep cleanup pending until the mutation settles, then remove all state before success. If deletion succeeds first, an enforced backend fence must reject or neutralize the released write without any post-success recreation. Inspect inventory and mutation history before repeated delete or corrective cleanup, with a separate live ownership control. Use bounded deadlines and authoritative settlement events; elapsed time and absence reads do not prove completion. The shared [TC-DELETE-02](../OSAC-1433-network-manager-integration-contract-networking/testplan.md#tc-delete-02) defines the safe ordered/fenced and deliberately unsafe oracle fixtures.

Other required traces cover target bind versus unbind; fixed conditional patch versus a later policy revision; and process loss before send, after send and after commit before acknowledgement. Repeat for Namespace, CUDN, policy and membership, including repeated deletion and foreign/new-owner controls.

An unproven ordering or fencing claim fails the relevant case. Pending cleanup is distinct from successful deletion; after releasing the held operation, cleanup must complete within the configured deadline. Indefinite pending state is not evidence of restart recovery. No executable tests, installed-cluster admission checks, packet tests or crash-recovery experiments were performed for this review note.
