# Defect — CAEO writes PAUSED / BLOCKED_BY_SUPPLY on every rescue, ignoring the connection's actual landing state

**ID:** C1
**Service:** `csp-customer-access-service` (CAEO)
**Repo / branch:** `wiom-tech/csp-os-yaml` @ `main` (verified at `d6171d9d5`)
**Severity:** High — customer-facing. A customer with a live, paid plan is left access-blocked.
**Status:** Open — raised by PM (Jaivin Prajapati), needs eng owner
**Found via:** design review for the *Customer-initiated pickup withdrawal* change spec. The defect is **pre-existing** and already reachable through the live recharge-rescue flow; the new feature makes it hit on **100%** of its cases rather than a subset.

---

## 1. Summary

`handleClConnectionRescued` unconditionally writes `cl_state = "PAUSED"` and transitions `caeo_state → CAEO_STATE_BLOCKED_BY_SUPPLY`, regardless of the state the connection actually landed in.

`CL_CONNECTION_RESCUED` carries `new_state`, which is **`ACTIVE` or `PAUSED`** depending on the recharge window. CAEO reads `new_state` exactly once in the entire handler — at the reactive-obligation gate on line 316. The two state writes above it never consult it.

Consequence: when a connection is rescued to **ACTIVE**, CAEO records it as PAUSED and blocks the customer's access.

---

## 2. Location

`services/csp-customer-access-service/src/main/java/io/wiom/csp/customer_access_entitlement/application/impl/InboundEventProcessingServiceImpl.java`

| Line | Code | Problem |
|---|---|---|
| 292 | `entity.setClState("PAUSED");` | Hardcoded. Should mirror `event.newState()`. |
| 294–305 | `if (currentState != BLOCKED_BY_SUPPLY) { … transition(currentState, BLOCKED_BY_SUPPLY) … }` | Runs on the ACTIVE path too. Sets `supply_disruption_start` and emits `CAEO_ACCESS_STATE_CHANGED(T4, SupplyState.PAUSED)`. |
| 316–323 | `if ("PAUSED".equals(event.newState()) && entitlementEnd > now)` | **Correct — do not change.** The only place `new_state` is honoured. |

Producer side, for reference — `csp-connection-lifecycle-service/.../InboundEventProcessingServiceImpl.java:1277-1309`:

```java
Instant windowEnd = rechargeWindowQueryPort.getActiveWindowEnd(...)
        .orElse(connection.getLatestRechargeWindowEnd());
boolean windowFuture = windowEnd != null && windowEnd.isAfter(now);
ConnectionState targetState = windowFuture ? ConnectionState.ACTIVE : ConnectionState.PAUSED;
...
eventPublisher.publishEvent(new ClConnectionRescued(..., targetState.name(), ...));
```

---

## 3. Worked example — what the customer actually experiences

Meet **Ramesh**. He has a Wiom connection with a **30-day plan he paid for on 1 July**, valid until **31 July**. His CSP's ISP recharge window is also open until **31 July**. Today is **10 July**.

| Time | What happens | Ramesh's experience |
|---|---|---|
| 10 Jul, 11:00 | Ramesh is browsing the app and taps **"Return device"** by mistake. | Internet still working. |
| 10 Jul, 11:00 | System: connection ACTIVE → PENDING_DEACTIVATION. Device marked awaiting recovery. Pickup task appears in the CSP's feed. CAEO records `cl_state = PENDING_DEACTIVATION` but **correctly leaves access ENABLED** — Ramesh has paid until 31 July. | Internet still working. Correct. |
| 10 Jul, 11:05 | Ramesh realises his mistake and recharges ₹300 to stop the pickup (today's only option). | Expects everything back to normal. |
| 10 Jul, 11:05 | Recharge triggers the rescue. CL checks the ISP window — **31 July, still open** — so the connection lands **ACTIVE**. Device restored to deployed. Pickup task cancelled. All correct. | So far so good. |
| 10 Jul, 11:05 | **← THE BUG.** CAEO receives the rescue and, ignoring that the connection landed ACTIVE, writes `cl_state = PAUSED` and flips access **ENABLED → BLOCKED_BY_SUPPLY**. | **Internet stops working.** |

**Ramesh's net position:** he paid ₹300 he did not need to pay, to undo a mistaken tap — and his internet is now *off*, on a plan valid for another 21 days. The connection record says ACTIVE. The device says deployed. Only the access record says blocked, and it is the one that gates his service.

Nothing errors. No alert fires. The only way anyone finds out is Ramesh calling support.

**And it leaves a second, delayed landmine.** His access record now says `cl_state = PAUSED` while the connection lifecycle says `ACTIVE`. The next time Ramesh recharges, the rescue gate reads that stale mirror, mis-evaluates, and **silently does not fire** — which is exactly how 3,084 connections got stuck in June.

### The same example, once fixed

Everything above holds until 11:05. Then CAEO reads the landing state, sees **ACTIVE**, and sets access to **ENABLED** with no supply-disruption marker. Ramesh's internet keeps working, and his mirror stays consistent so his next recharge behaves.

### The contrast case — where today's behaviour is right

Same setup, but Ramesh's **CSP** has not paid the ISP, so the ISP window lapsed on **5 July**. Ramesh's own plan is still live to 31 July. On rescue, CL lands the connection **PAUSED** (supply genuinely is down). Here CAEO *should* write `PAUSED` / `BLOCKED_BY_SUPPLY` and raise a reactive obligation against the CSP — and today it does. **This path is correct and must not change.** The fix is only about telling the two cases apart.

---

## 4. Reproduction

1. Connection ACTIVE, plan live. `caeo_state = CAEO_STATE_ENABLED`, `cl_state = "ACTIVE"`.
2. Deactivation is initiated (customer pickup request, or plan-expiry auto-deactivation). `handleClDeactivationInitiated` (`:240-266`) sets `cl_state = "PENDING_DEACTIVATION"` and **deliberately leaves `caeo_state` untouched** — still ENABLED. Correct.
3. Connection is rescued while the recharge window is still in the future. CL transitions PENDING_DEACTIVATION → **ACTIVE** and emits `CL_CONNECTION_RESCUED` with `new_state = "ACTIVE"`.
4. CAEO `handleClConnectionRescued` runs:
   - `cl_state` ← `"PAUSED"` (CLOS says ACTIVE)
   - `caeo_state` ← `CAEO_STATE_BLOCKED_BY_SUPPLY` (was ENABLED)
   - `supply_disruption_start` ← now
   - emits `CAEO_ACCESS_STATE_CHANGED(T4, SupplyState.PAUSED)`

**Observed:** customer is access-blocked with a live, paid plan.
**Expected:** `cl_state = "ACTIVE"`, `caeo_state = CAEO_STATE_ENABLED`, no supply-disruption marker, no T4 emission.

---

## 5. Why this is worse than a normal state bug

**5.1 It fails silently.** `ENABLED → BLOCKED_BY_SUPPLY` is a legal T4 transition (`domain/service/CaeoStateMachine.java:14-18`), so `CaeoStateMachine.transition()` does not throw. The write commits. There is no exception, no failed event, no DLQ entry — only a log line that reads as normal operation.

**5.2 It re-creates the `cl_state` mirror drift that has already caused a production incident.** CAEO's rescue-emit gate reads the `cl_state` **mirror**, not live CLOS. Every connection that goes through this path is left with `cl_state = "PAUSED"` while CLOS holds `ACTIVE` — so on that customer's *next* recharge the gate mis-evaluates and the rescue silently does not fire. This is the same failure mode as the June 2026 incident where 3,105 `cl_state` rows had to be corrected by hand (3,084 → `PENDING_DEACTIVATION`) after rescues stopped firing. This defect quietly re-seeds that dataset.

**5.3 Blast radius is wider than the new feature.** Any rescue landing on ACTIVE is affected. That includes the existing recharge-rescue path whenever the customer recharges while a live recharge window is still open — which is the common case for an early recharge.

---

## 6. Proposed fix

Branch both writes on `event.newState()`, leaving the obligation gate untouched.

```java
boolean landedActive = "ACTIVE".equals(event.newState());
entity.setClState(landedActive ? "ACTIVE" : "PAUSED");

CaeoState target = landedActive
        ? CaeoState.CAEO_STATE_ENABLED
        : CaeoState.CAEO_STATE_BLOCKED_BY_SUPPLY;

if (currentState != target) {
    CaeoStateMachine.transition(currentState, target);
    entity.setPriorAccessState(currentState);
    entity.setCaeoState(target);
    if (!landedActive) {
        entity.setSupplyDisruptionStart(clockService.now());
    } else {
        entity.setSupplyDisruptionStart(null);   // clear stale marker
    }
    updateTraceability(entity);
    stateRepository.save(entity);
    emitAccessStateChanged(entity, currentState, target,
        landedActive ? TransitionId.T5a : TransitionId.T4,
        event.eventId(),
        landedActive ? SupplyState.ACTIVE : SupplyState.PAUSED);
} else {
    updateTraceability(entity);
    stateRepository.save(entity);
}

// lines 314-323 unchanged
```

Notes for the implementer:

- `BLOCKED_BY_SUPPLY → ENABLED` is already legal as **T5a** (`CaeoStateMachine.java:24-28`), so the ACTIVE path needs no state-machine change.
- `ENABLED → ENABLED` is not in the transition map; the `currentState != target` check must guard the call (it already does in the current shape — preserve it).
- Confirm the correct `TransitionId` / `SupplyState` pairing for the ACTIVE path against the CAEO OS spec before merging. T5a is the inferred match, not a verified one.
- **Do not touch lines 314–323.** The obligation gate (`new_state == PAUSED` **AND** `entitlement_end > now`) is correct as written and matches the intended rule: raise a reactive obligation only when the ISP window has lapsed while the customer's enrichment is still live.

---

## 7. Suggested test cases

| # | Given | When | Then |
|---|---|---|---|
| 1 | `caeo_state = ENABLED`, `cl_state = PENDING_DEACTIVATION`, recharge window in future | `CL_CONNECTION_RESCUED(new_state=ACTIVE)` | `cl_state = ACTIVE`, `caeo_state = ENABLED`, no `supply_disruption_start`, no obligation |
| 2 | `caeo_state = ENABLED`, `cl_state = PENDING_DEACTIVATION`, window lapsed, entitlement future | `CL_CONNECTION_RESCUED(new_state=PAUSED)` | `cl_state = PAUSED`, `caeo_state = BLOCKED_BY_SUPPLY`, `supply_disruption_start` set, obligation created — **regression guard, current behaviour** |
| 3 | `caeo_state = BLOCKED_BY_SUPPLY`, stale `supply_disruption_start` set | `CL_CONNECTION_RESCUED(new_state=ACTIVE)` | `caeo_state = ENABLED`, `supply_disruption_start` cleared |
| 4 | Window lapsed, entitlement **also** lapsed | `CL_CONNECTION_RESCUED(new_state=PAUSED)` | State writes as case 2; **no** obligation created — **regression guard** |
| 5 | `caeo_state = CLOSED` | any `CL_CONNECTION_RESCUED` | No-op, early return — **regression guard** |
| 6 | Same event delivered twice | replay | Idempotent; no duplicate `CAEO_ACCESS_STATE_CHANGED` |

**Invariant to assert across all paths:** CAEO `cl_state` always equals the CLOS `current_state` for that connection immediately after a rescue is processed.

---

## 8. Remediation of existing data

Connections already rescued onto the ACTIVE path carry the wrong state today. Before/alongside the fix, worth a read-only audit:

- CAEO rows where `cl_state = 'PAUSED'` **and** CLOS `current_state = 'ACTIVE'` — the drifted mirror set.
- Of those, rows with `caeo_state = 'CAEO_STATE_BLOCKED_BY_SUPPLY'` and `entitlement_end > now()` — customers currently access-blocked while paid. These are the ones to correct first.

Sizing and the correction script are eng's call; flagging that a data fix is likely needed in addition to the code fix.

---

## 9. Relationship to the pickup-withdrawal change spec

The change spec **Customer-initiated pickup withdrawal (CSP side)** depends on this fix. That feature's eligibility rule requires the customer's plan to be live, so it always lands on the ACTIVE branch — meaning it would hit this defect on every single execution. The spec documents this as a named prerequisite and must not start build until C1 is merged.

C1 is filed separately because it is not caused by that feature and should be fixed on its own timeline regardless of whether the feature ships.
