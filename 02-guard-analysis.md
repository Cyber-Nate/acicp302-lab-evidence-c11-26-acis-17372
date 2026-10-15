# Supplied Defensive Guard Analysis (`guard.py`)

This analysis covers the guard already included in the course source. No new
guard was built; every check and limitation described below is traced to a
specific point in `plant_sim/guard.py`, not assumed.

## 1. What the guard is

`guard.py` runs a second, independent Modbus/TCP server on `127.0.0.1:5021`.
It is **not** a modification of the plant's own server - it is a separate
process that holds its own local register copy (`GuardedContext`, a subclass
of `ModbusSlaveContext`) and, on an accepted write, forwards that write on to
the real plant on port 5020 using its own `ModbusTcpClient`. A client is only
protected if it chooses to connect to port 5021; nothing prevents a client
from connecting to port 5020 directly (see limitation L-04).

## 2. Decision table

Each incoming write to the guard (`GuardedContext.setValues`) is evaluated
against up to three checks, in this order, for registers 0–2 only
(`guard._check`):

| Check | Applies to | Condition (as implemented) | Decision if triggered | Event/log evidence |
|---|---|---|---|---|
| Absolute range | Registers 0 (temp), 1 (pressure), 2 (flow) | `requested` value outside the register's `*_RANGE` tuple (temp 600–750, pressure 300–400, flow 500–1500, all tenths) | Write rejected. Nothing forwarded to the plant, nothing written to the guard's local mirror, drift history not updated | `log_event()` appends a JSON line to `plant_sim/events.jsonl` with a reason string such as `"95.0bar outside safe range 30.0-40.0bar"` |
| Step limit | Registers 0–2 | `abs(requested - current)` exceeds `MAX_TEMP_STEP` (30 → 3.0 °C), `MAX_PRESSURE_STEP` (50 → 5.0 bar) or `MAX_FLOW_STEP` (100 → 10.0 Nm³/h) in a single write, where `current` is the guard's own last-accepted register value | Write rejected, same as above | Reason string such as `"step of 11.0C exceeds max 3.0C"` |
| Rolling-baseline drift | Registers 0–2 | Evaluated once the history holds 3 accepted values for that register (`DRIFT_WINDOW = 3`); the background sync thread also appends each safe mirrored reading to this history every second, so in normal operation it is always full. Compares the newly requested value to the accepted value from exactly 3 accepted writes ago; rejects if the difference exceeds `DRIFT_THRESHOLD_TEMP` (20 → 2.0 °C), `DRIFT_THRESHOLD_PRESSURE` (20 → 2.0 bar) or `DRIFT_THRESHOLD_FLOW` (40 → 4.0 Nm³/h) | Write rejected, same as above | Reason string such as `"cumulative drift of 3.0bar over last 3 accepted writes exceeds 2.0bar limit"` |
| (no check) | Any register other than 0, 1, 2 (e.g. register 3, the alarm flag) | - | **Write is forwarded to the plant unconditionally, with no validation at all** | No event is logged for this path - `log_event()` is only called from the rejection branch |

If, and only if, a write to register 0–2 passes all applicable checks, the
guard: forwards the write to the real plant via `plant_client.write_register`,
updates its own local mirror via `super().setValues(...)`, and appends the
accepted value to that register's rolling-history `deque` (`maxlen=3`).
Rejected writes update none of these three things - the guard's exposed state
simply does not move.

## 3. The passive-mirroring safeguard (`sync_safe_reads`)

Separately from the write-path checks above, a background thread polls the
real plant's registers 0–3 once per second and copies them into the guard's
own store, **but only if temperature, pressure and flow are all simultaneously
within their absolute safe ranges at that instant**. This means:

- If an attacker bypasses the guard entirely and writes an unsafe value
  straight to the plant on port 5020, the guard's own mirrored state does
  **not** pick up that unsafe value - it simply stops updating from the plant
  until the plant returns to a fully-safe state. The guarded dashboard appears
  to freeze rather than show the unsafe reading.
- This is a genuine protective behaviour (bad data cannot silently flow
  through to the guarded view), but it is also a limitation: freezing is not
  the same as raising a positive alert that something is wrong on the direct
  path (see L-07).

## 4. Assumptions behind the checks

- The three checks assume an attacker moves values either in one large step
  (caught by range/step) or many small steps (caught by rolling drift), but
  the drift window compares only to the state **3 accepted writes ago**, not
  to a long-term or original setpoint - see L-08.
- The checks assume all legitimate process change arrives as single-register
  writes through this guard process; they have no model of what a
  legitimate *operator* command sequence should look like beyond these three
  numeric bounds.

## 5. Guard limitations (verified from source, not inferred)

| ID | Limitation | Source evidence |
|---|---|---|
| L-01 | No client authentication or authorization on the guard's Modbus interface. Any process that can reach `127.0.0.1:5021` can issue reads and writes. | `guard.py` - `StartTcpServer` is called with no auth configuration; `pymodbus` default server has none enabled here |
| L-02 | The guard validates **writes only**. `GuardedContext` overrides `setValues()` but never overrides `getValues()`, so reads pass straight through to the underlying `ModbusSequentialDataBlock` unmodified. Reconnaissance/enumeration (e.g. `recon_scan.py`) is just as unrestricted through the guard as through the plant directly. | `guard.py` - no `getValues` override present in `GuardedContext` |
| L-03 | Only registers 0–2 are validated. A write to any other register address, including register 3 (the alarm flag), falls into the `else` branch of `setValues()` and is forwarded to the real plant with **zero** validation. | `guard.py`, `GuardedContext.setValues`, final `else` branch |
| L-04 | The guard cannot force traffic through itself. A client that simply connects to port 5020 instead of 5021 bypasses the guard completely; nothing in the architecture prevents this choice. | `docs/reference-architecture.md`; `plant.py` listens independently on 5020 with no gating by the guard |
| L-05 | No anti-replay or freshness protection. All three numeric checks are value-based, not sequence/identity-based, so an already-accepted, still-in-range value can be resent any number of times and will be accepted every time. | `guard.py`, `_check()` - no nonce, timestamp or write-counter comparison anywhere in the method |
| L-06 | No rate limiting or connection throttling on either the plant or the guard, so nothing in the source defends against a high-volume read (or write) flood. | `plant.py`, `guard.py` - no limiter in either `main()` |
| L-07 | The guard's safe-mirroring behaviour (Section 3) prevents bad data from propagating to the guarded view, but produces a silent freeze rather than a positive alert that the underlying plant is currently unsafe. | `guard.py`, `sync_safe_reads()` |
| L-08 | The rolling-drift baseline only ever compares against the value from exactly 3 accepted entries ago, and the guard's own sync thread adds a mirrored plant reading to that history every second, so the window spans only a few seconds. An attacker who lands a few small accepted steps, then pauses long enough for legitimate background drift to refresh the window, could in principle still accumulate gradual creep across multiple separated attack sessions. | `guard.py`, `DRIFT_WINDOW = 3` and `_check()` drift comparison logic; SV-03 complete event log (drift did not fire for requests of 40.2 and 41.2 bar and first fired at 42.2 bar) |
| L-09 | This is explicitly an educational, rule-based example (per the course's own `docs/reference-architecture.md`), not a certified process-safety or security product; it has no redundancy, no independent safety-instrumented layer, and no tamper protection on `events.jsonl` itself. | `docs/reference-architecture.md`, safety note |
| L-10 | Design observation: the guard's accepted range (30.0 to 40.0 bar) overlaps the plant's high-pressure alarm band (alarm at 39.5 bar and above), so the guard accepts values that put the process into alarm. Observed: the guarded recon read showed alarm = 1 with pressure at 39.5 bar while the process sat near the top of its range. | `plant.py` alarm condition (`pressure >= PRESSURE_RANGE[1] - 5`), `guard.py` `PRESSURE_RANGE`, SV-05 guarded output |

## 6. Mapping to risk register

L-01 through L-10 above are carried forward into `04-risk-register.csv` as
R-04 through R-09 (plus the unauthenticated-access findings R-01/R-02/R-03
covering the plant, guard and dashboards generally).

## 7. Confirmation against the live run (2026-10-15, VM clock)

Genuine guard event lines from the run. The complete log is `evidence/05-scenarios/events-full.jsonl` (103 events, 05:52:54 to 06:02:03),
and its counts reconcile with the console output: 74 events for the 74 attack_hold writes, 18 for the 6 x 3 false_data writes, and 11 for
slow_drift steps 2 to 12 (step 1 was accepted).

```
SV-01 (guarded attack_hold), one of 74 identical-pattern events:
{"timestamp": "2026-10-15 05:53:02", "reason": "95.0 bar outside safe range 30.0-40.0 bar; step of 57.5 bar exceeds max 5.0 bar; cumulative drift of 57.7 bar over last 3 accepted writes exceeds 2.0 bar limit"}

SV-02 (guarded false_data_attack), two of 18:
{"timestamp": "2026-10-15 05:57:50", "reason": "110.0C outside safe range 60.0-75.0C; step of 35.7C exceeds max 3.0C; cumulative drift of 35.8C over last 3 accepted writes exceeds 2.0C limit"}
{"timestamp": "2026-10-15 05:57:50", "reason": "200.0 Nm3/h outside safe range 50.0-150.0 Nm3/h; step of 55.7 Nm3/h exceeds max 10.0 Nm3/h; cumulative drift of 56.1 Nm3/h over last 3 accepted writes exceeds 4.0 Nm3/h limit"}

SV-03 (guarded slow_drift_attack), first six of 11 (requested 40.2 up to 45.2 bar):
06:01:43 40.2 bar outside safe range 30.0-40.0 bar
06:01:45 41.2 bar outside safe range 30.0-40.0 bar
06:01:47 42.2 bar outside safe range 30.0-40.0 bar; cumulative drift of 2.3 bar over last 3 accepted writes exceeds 2.0 bar limit
06:01:49 43.2 bar outside safe range 30.0-40.0 bar; cumulative drift of 3.3 bar over last 3 accepted writes exceeds 2.0 bar limit
06:01:51 44.2 bar outside safe range 30.0-40.0 bar; cumulative drift of 4.2 bar over last 3 accepted writes exceeds 2.0 bar limit
06:01:53 45.2 bar outside safe range 30.0-40.0 bar; step of 5.3 bar exceeds max 5.0 bar; cumulative drift of 5.2 bar over last 3 accepted writes exceeds 2.0 bar limit
```

**Confirmed by the run**

- The range, step and drift checks each fired, and their wording in the log matches the `_check` logic. In SV-03 the range check alone blocked
  the requests at 40.2 and 41.2 bar, drift joined it from 42.2 bar (2.3 bar), and step joined from 45.2 bar (a step of 5.3 bar against the 5.0 bar limit).
- Rejected writes were dropped entirely: the guarded readings never left the safe range (0 of 74 in SV-01, 0 of 6 per channel in SV-02), and every
  write in SV-01 and SV-02 produced exactly one logged rejection.
- The guard validates writes only: SV-05 returned registers 0 to 49 through port 5021 exactly as through port 5020.
- No anti-replay check was observed: after the SV-04 replays the complete event log gained no BLOCKED line (indirect evidence, since the guard logs only rejections).

**Differences from this analysis as first written (recorded, not edited away)**

1. **Reasons are combined.** `_check` returns every violated check and `setValues` joins them with "; ", so a single event usually lists range,
   step and drift together. The examples in the decision table show one check per reason for clarity.
2. **The drift history is always populated.** The drift reason appeared on the very first attack write in SV-01 (57.7 bar), because the sync thread
   keeps the history full with mirrored plant readings. The decision table originally said the drift check waits for 3 accepted writes; it has been corrected above.
3. **SV-03 did not isolate the drift check, and the log shows it lagging.** The run started at 38.2 bar, so the absolute-range check blocked from the
   second step. The drift check stayed silent for the requests at 40.2 and 41.2 bar (under the 2.0 bar threshold) and first fired at 42.2 bar, when the request was
   already above the safe ceiling. This supports L-08: a +1.0 bar per 2 s ramp is not stopped by the drift check before the absolute limit does it.
