# Proposal: Age-based node rotation via drift

- **Status**: Provisional
- **Authors**: @charlie-e2b
- **Created**: 2026-11-02
- **Related Issues**: none

---

## Summary

Add an optional `spec.maxNodeAge` duration to `GCENodeClass`. The cloud provider's drift check reports a NodeClaim older than that as drifted with reason `AgeDrift`. Karpenter core then handles it like any other drift: it launches a replacement, waits for it to initialize, taints the old node, and drains it within the NodePool's `terminationGracePeriod`, all under the NodePool's disruption budgets.

This gives operators a scheduled rotation that never reduces capacity, which the NodePool's `expireAfter` cannot do: since Karpenter v1, expiry is forceful. It deletes the NodeClaim the moment the age is reached, before a replacement exists, and does not consult budgets.

---

## Motivation

### Problem Statement

Rotating nodes on a schedule picks up new node images and clears slow leaks. Karpenter offers `expireAfter` for this, but its v1 semantics are a bad fit for workloads with long drains. An expired node is tainted and its pods evicted immediately; the replacement only appears when the evicted pods become pending, and with a PodDisruptionBudget or `do-not-disrupt` that can be as late as `terminationGracePeriod`. For a pool where each node hosts a service that needs up to a day to drain, the pool runs one node short for that whole day on every rotation.

Drift has the ordering operators want: replacement first, then disruption, bounded by budgets. Nothing today lets drift fire on age.

### Goals

- Scheduled, per-node rotation that launches the replacement before disrupting the old node and respects disruption budgets.
- Spread rotations of a fleet created together rather than firing them all at once.
- Zero behaviour change when the field is unset.

### Non-Goals

- Replacing or changing `expireAfter`.
- Making the rotation point configurable beyond the duration itself; the jitter rule is fixed.

---

## Proposal

### Overview

```yaml
apiVersion: karpenter.k8s.gcp/v1alpha1
kind: GCENodeClass
spec:
  maxNodeAge: 336h
```

### Design Details

- `GCENodeClassSpec.MaxNodeAge *metav1.Duration`, JSON `maxNodeAge`, same string pattern as `preemptionNoticeDuration`. Tagged `hash:"ignore"`, so changing the policy does not itself drift every node; the new value applies at the next drift check.
- `isNodeClassDrifted` gains a third check after static-field and image drift: `isAgeDrifted` returns `AgeDrift` when `now - nodeClaim.creationTimestamp >= threshold`.
- `threshold = maxNodeAge - jitter`, where `jitter` is derived from an FNV hash of the NodeClaim name and lies in `[0, maxNodeAge/10)`. Each node therefore rotates at a fixed point between 90% and 100% of `maxNodeAge`, and never later than `maxNodeAge`.
- Core re-evaluates drift every five minutes, so rotation lands within that window of the threshold.
- No change to `expireAfter`; both can be set, in which case whichever fires first applies.

---

## Alternatives Considered

- **Graceful expiry in karpenter core.** The right long-term home, since it is provider-neutral, but a core change with a release cycle this provider does not control. The provider-level drift reason is small and can be removed if core grows the option.
- **Periodic NodePool template label bump.** Works today with no code, but rotates the whole pool on each bump rather than each node by its own age, and needs an external cron.

---

## Acceptance Criteria

- A node older than `maxNodeAge` gets the `Drifted` condition with reason `AgeDrift`, and its replacement is initialized before the node is tainted.
- Rotation honours `budgets` and `terminationGracePeriod`.
- Nodes created together rotate at distinct times within the last 10% of `maxNodeAge`.
- A `GCENodeClass` without the field is unchanged, and changing the field does not drift nodes by itself.
