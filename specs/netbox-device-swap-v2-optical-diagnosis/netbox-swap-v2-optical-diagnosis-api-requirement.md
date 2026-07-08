# Netbox Device Swap V2 — Optical Diagnosis Gate: API / Data Requirement

**Purpose:** Define exactly what data Wiom's swap layer needs from the telemetry/remote team so the
**Optical Diagnosis Gate** can decide whether a device swap should be *blocked* (the problem is optical /
network, not the device) or *allowed*.

**Audience:** Telemetry-Remote team (data source owner) + ACS team (consumer/orchestrator).

**Important:** The logic below is validated today via Metabase / Snowflake on the
`HOURLY_DEVICE_PING_INFLUX` table (Database 113). **We will NOT ship on the Metabase source.** The
production gate must call a proper API. This doc specifies the data contract that API must satisfy. The
Metabase table is used here only to (a) prove the logic and (b) give real example payloads.

---

## 1. Where this sits in the swap flow

V1 already runs a gate-check phase (G1–G11) before the physical-handoff screen — pass/fail, no record
created, no reservation. The Optical Diagnosis Gate is a **new gate in that same phase**, evaluated
**after** the custody/state gates (G1–G5) pass and **before** the CSP is shown the physical-handoff
screen.

It behaves exactly like G1–G11: **hard block**. On block the CSP cannot move forward; they are shown a
message (frontend copy TBD — placeholder) telling them what to fix and to contact Wiom support if needed.

---

## 2. Locked decisions (do not re-open)

| # | Decision |
|---|----------|
| D1 | **Fail-open on no data.** If no valid optical data exists for the old device, **skip** the optical check and **allow** the swap — insufficient evidence to blame device vs network. |
| D2 | **Strict ops logic — no additions.** Only the three block conditions and the two exclusion rules below. Nothing extra. |
| D3 | **Old device only.** The check runs on the `old_device_id` (the device being swapped out). New device is not checked here. |
| D4 | **Hard block.** No CSP override at this gate (unlike the post-swap validation layer). Blocked = cannot proceed; CSP contacts Wiom support. |
| D5 | **Applies to every fault type.** DEVICE_HARDWARE, DEVICE_SOFTWARE, DEVICE_CONNECTIVITY — the gate runs regardless of the diagnosed fault type. |
| D6 | **Block = restart from zero.** After a hard block, if the CSP exits and returns, the swap flow re-enters from the beginning (re-runs all gates). No resumable state is created at a blocked gate. |
| D7 | **`OPTICAL_AVG = 0` is a missing-data sentinel.** It is excluded from inputs. If exclusions leave zero valid rows, that counts as "no valid data" → per D1, allow the swap (bypass, because Wiom could not reliably confirm). |
| D8 | **Wiom holds the raw data.** ACS may compute the verdict on their side or Wiom's — team's choice — but the raw hourly rows must be available to Wiom's layer so gate thresholds can be tuned independently later without a telemetry-team release. |

---

## 3. The diagnosis logic (exact)

### 3.1 Input filtering — keep only valid optical readings

For the old device, over the lookback window, take each hourly record and **exclude** it if **any** of:

- `OPTICAL_AVG` is `NULL`
- `OPTICAL_AVG` = `0`  *(sentinel for "no optical reading", see D7)*
- `TOTAL_PINGS_RECEIVED` = `0`

Everything remaining is a **valid reading**. Each valid record contributes its `OPTICAL_AVG` value.

### 3.2 Block conditions

Compute across the valid readings:

- `avg_optical`  = average of `OPTICAL_AVG`
- `max_optical`  = maximum of `OPTICAL_AVG`
- `min_optical`  = minimum of `OPTICAL_AVG`

**Block the swap if ANY of:**

| Condition | Meaning |
|-----------|---------|
| `avg_optical < -25` dBm | Signal too weak → fiber / optical / network side, not the device |
| `avg_optical > -8` dBm  | Signal too strong / out of range → optical side, not the device |
| `(max_optical − min_optical) > 3` dBm | Unstable / flapping optical signal → network side |

If none of the three are true → **allow** the swap.

### 3.3 No valid data

If §3.1 leaves **zero** valid readings → **skip** the check and **allow** the swap (D1 / D7).

> **Note on "readings":** each valid hour contributes its `OPTICAL_AVG`. The block-range check therefore
> uses max/min of the hourly `OPTICAL_AVG` series. The table also carries per-hour `OPTICAL_MIN` /
> `OPTICAL_MAX` (intra-hour extremes) — not used by the strict ops logic, but available if we ever want a
> tighter instability check.

### 3.4 Lookback window

The three block conditions are unambiguous. The window over which valid hourly readings are gathered is a
tunable parameter, fixed for V1 of the spec:

| Parameter | Meaning | Value |
|-----------|---------|-------|
| `P_OPTICAL_LOOKBACK` | Time window over which valid hourly readings are gathered | **1 hour** (tunable; may change on ops input) |

With a 1-hour window the check reads the **latest available hourly optical record** for the old device
and evaluates it. If that hour has no valid reading (excluded per §3.1), it is treated as no-data → allow
(D1 / D7).

---

## 4. Source table reference (`HOURLY_DEVICE_PING_INFLUX`, DB 113)

The API must expose, per old device per hour in the window, the fields Wiom needs. From the live table,
the relevant columns are:

| Column | Type | Needed? | Use |
|--------|------|---------|-----|
| `DEVICE_ID` | VARCHAR | ✅ | Match to `old_device_id` |
| `HOUR_START_IST` | TIMESTAMP | ✅ | Bound the lookback window / order the series |
| `TOTAL_PINGS_RECEIVED` | NUMBER | ✅ | Exclusion filter (drop hour if `= 0`) |
| `OPTICAL_AVG` | NUMBER (signed dBm) | ✅ | The reading; drives all three block conditions; exclusion filter (drop if `NULL` or `0`) |
| `OPTICAL_MIN` | NUMBER (signed dBm) | ➖ optional | Not in strict logic; useful for future intra-hour instability check |
| `OPTICAL_MAX` | NUMBER (signed dBm) | ➖ optional | Same as above |

All optical values are **signed dBm** (negative). "Less than −25" = more negative / weaker; "greater
than −8" = less negative / stronger.

Full column list of the table (for reference): `PARTNER_ID, DEVICE_ID, NAS_ID, DATE_IST, HOUR_START_IST,
HOUR_END_IST, TOTAL_PINGS_RECEIVED, TOTAL_PINGS_MISSED, FRAGMENTED_PING_MISSES,
CONTINUOUS_MISSED_PING_INSTANCES, MAX_PINGS_MISSED_IN_CONTINUOUS_INSTANCE,
MIN_PINGS_MISSED_IN_CONTINUOUS_INSTANCE, FIRST_MISS_TIMESTAMP_IST, LAST_MISS_TIMESTAMP_IST, OPTICAL_MIN,
OPTICAL_AVG, OPTICAL_MAX, CONNECTED_2G_MAX, CONNECTED_5G_MAX, CONNECTED_WIOMNET_MAX, FIRST_PING_TS_IST,
LAST_PING_TS_IST, PING_BITMAP, BANDSTEERING_FLAG, SSID_JSON, UPDATED_AT, SOURCE_TABLE, INSERTED_AT,
OPTICAL_IN_RANGE_PINGS`.

---

## 5. Proposed API contract

### 5.1 Request (Wiom → telemetry/remote API)

```
GET /device/optical-history?device_id=<old_device_id>&lookback_hours=<P_OPTICAL_LOOKBACK>
```

- `device_id` — the **old** device being swapped out.
- `lookback_hours` — window size (tunable; see §3.4).

### 5.2 Response — minimum viable (raw rows; Wiom computes the verdict)

This is the **preferred** shape per D8: the API returns raw hourly rows, Wiom applies exclusions +
thresholds. Keeps the API "dumb" and lets Wiom tune gate logic independently.

```json
{
  "device_id": "GX44909",
  "lookback_hours": 1,
  "records": [
    { "hour_start_ist": "2026-05-02T00:00:00+05:30", "total_pings_received": 12, "optical_avg": -21, "optical_min": -21, "optical_max": -21 },
    { "hour_start_ist": "2026-05-02T01:00:00+05:30", "total_pings_received": 12, "optical_avg": -21, "optical_min": -21, "optical_max": -21 },
    { "hour_start_ist": "2026-05-02T02:00:00+05:30", "total_pings_received": 0,  "optical_avg": null, "optical_min": null, "optical_max": null }
  ]
}
```

**Required per record:** `hour_start_ist`, `total_pings_received`, `optical_avg`.
**Optional per record:** `optical_min`, `optical_max`.

### 5.3 Response — alternative (pre-computed verdict)

If ACS/telemetry prefer to compute server-side, the API may instead return the decision. **Only
acceptable if** the raw rows remain retrievable by Wiom for tuning/audit (D8). The exclusion rules and
thresholds MUST match §3 exactly.

```json
{
  "device_id": "GX44909",
  "lookback_hours": 1,
  "valid_reading_count": 1,
  "avg_optical": -27,
  "min_optical": -27,
  "max_optical": -27,
  "optical_range": 0,
  "verdict": "BLOCK",
  "block_reason": "AVG_BELOW_-25"
}
```

`verdict` ∈ `BLOCK` / `ALLOW`. `block_reason` ∈ `AVG_BELOW_-25` / `AVG_ABOVE_-8` /
`RANGE_ABOVE_3` / `null`. When `valid_reading_count = 0` → `verdict = ALLOW`, reason `NO_VALID_DATA`.

---

## 6. Worked examples (real values from `HOURLY_DEVICE_PING_INFLUX`)

All rows below are actual telemetry pulled from DB 113 (date 2026-05-02).

### Example A — ALLOW (healthy signal)
`GX98531`, 12 valid hours, every hour `OPTICAL_AVG = -21`, pings healthy.
- avg = −21 (within −25…−8) ✓ · range = 0 (≤ 3) ✓ → **ALLOW**.

### Example B — BLOCK (signal too weak)
`GX44909`, `TOTAL_PINGS_RECEIVED = 1`, `OPTICAL_AVG = -27`.
- avg = −27 < −25 → **BLOCK** (`AVG_BELOW_-25`). Swap won't fix a weak-fiber problem.

### Example C — BLOCK (signal too strong)
`GX44893`, `TOTAL_PINGS_RECEIVED = 12`, `OPTICAL_AVG = -1`.
- avg = −1 > −8 → **BLOCK** (`AVG_ABOVE_-8`). Out-of-range optical, not a device fault.

### Example D — Excluded rows → treated as no-data → ALLOW
`GX78435` (and `GX75243`, `GX48220`, `PNM01257`, `GX122981`): `TOTAL_PINGS_RECEIVED = 12` but
`OPTICAL_MIN/AVG/MAX = 0/0/0`. `OPTICAL_AVG = 0` is the missing-data sentinel → **excluded** (D7). If a
device's entire window looks like this, valid_reading_count = 0 → **ALLOW** (bypass; Wiom could not
reliably confirm).

### Example E — BLOCK (unstable signal) — illustrative
A device whose valid hourly `OPTICAL_AVG` ranges from −19 (max) to −24 (min) over the window.
- range = (−19) − (−24) = 5 > 3 → **BLOCK** (`RANGE_ABOVE_3`). Flapping optical = network side.

---

## 7. Open items to close before build

1. **`P_OPTICAL_LOOKBACK`** — fixed at **1 hour** for this spec (§3.4); tunable if ops changes it.
2. **API ownership & endpoint** — telemetry/remote team to confirm the endpoint, auth, and latency
   (this gate runs synchronously inside the pre-swap gate-check, so it must respond fast).
3. **Raw vs computed** — ACS to choose §5.2 vs §5.3, subject to D8 (Wiom retains raw access).
4. **Frontend block message** — placeholder; copy owned by product/design.
