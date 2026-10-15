# ACICP302ICS Final Practical Report

## Cover information

- Course: ACICP302ICS - Network Security & Attacks
- Practical title: Analyze, Test & Defend the Supplied Modbus/TCP OT Simulation
- Learner: Nathaniel Temitope Ogunjobi
- Registration number: C11/26/ACIS/17372
- Submission date: 15 October 2026
- Learner evidence-repository URL and final commit: `[INSERT]`
- Course source URL: `https://github.com/icdfa/ACICP302ICS-Network-Security-and-Attacks`
- Course source commit used: `2a66f9bcf253eb3fe31c68ad52794cd535421da8`
- Assigned VM/snapshot: Ubuntu 64-bit guest on VMware Workstation (hostname cyber-nate-VMware-Virtual-Platform); snapshot: LAB-START-1

## 1. Executive summary

This report analyzes the course-supplied Modbus/TCP OT simulation at
`labs/ot-security/student-lab-source/`: a simulated plant process (`plant.py`)
exposing temperature, pressure and flow as Modbus/TCP holding registers with
no write validation, and a separate validation proxy (`guard.py`) offering an
alternative, protected path that applies absolute-range, step and
rolling-baseline-drift checks before forwarding writes to the real plant.

Working entirely on loopback (`127.0.0.1`) inside the assigned Ubuntu VM, five
approved demonstration scenarios were run against both the direct (port 5020)
and guarded (port 5021) paths. On the direct path every out-of-range write reached the plant (attack_hold: 77 of 77 read-backs unsafe; false_data_attack: 6 of 6 on all three channels). On the guarded path none did (0 of 74; 0 of 6 per channel) and each was dropped and logged with its reasons. The guard raised no objection to repeated identical in-range writes (replay) or to full register reads (recon). The slow-drift run was contained by the guard's absolute-range check rather than its drift check, and parts of the pre-run prediction for that scenario were corrected (Section 7).

All four services were stopped with Ctrl+C at the end of the session, and a port check at 11:25 confirmed that nothing was listening on ports 5020, 5021, 8080 or 8081.

## 2. Scope, safety and method

- Assigned VM: Ubuntu 64-bit guest on VMware Workstation (hostname cyber-nate-VMware-Virtual-Platform)
- Canonical source path used: `labs/ot-security/student-lab-source/` only; `labs/ot-security/archive-original/` was not executed
- Authorized scope: loopback (`127.0.0.1`) only; the VM's network adapter was disconnected for the lab run, per the Student Lab Guide
- No real or external system was targeted at any point
- Evidence collection method: Wireshark capture on the loopback (`lo`) interface filtered to the lab's Modbus/TCP port; screenshots of both dashboards; console output from each demonstration script; relevant lines from `plant_sim/events.jsonl`

## 3. Supplied architecture and process model

See `01-topology-and-register-map.md` for the full annotated topology,
component inventory and verified register map. In summary: `plant.py` is an
unprotected Modbus/TCP server on port 5020; `guard.py` is a second, separate
server on port 5021 that validates writes to registers 0–2 before forwarding
them to the plant, and separately mirrors safe plant telemetry into its own
store once per second. Two HTTP dashboards (`dashboard.py` on 8080,
`dashboard_guarded.py` on 8081) provide the direct and guarded views
respectively.

The verified register map (full detail in `01-topology-and-register-map.md`,
Section 4):

| Register | Point | Unit/scale | Safe range |
|---|---|---|---|
| 0 | Temperature | tenths of °C | 60.0–75.0 °C |
| 1 | Pressure | tenths of a bar | 30.0–40.0 bar |
| 2 | Flow | tenths of Nm³/h | 50.0–150.0 Nm³/h |
| 3 | High-pressure alarm (bool) | - | set when pressure ≥ 39.5 bar |

## 4. Normal operations and packet evidence

Baseline readings across the session stayed inside the safe ranges (temperature 65-72 C, pressure 30-33 bar, flow 81-102 Nm3/h on the dashboards and logs) with no alarm. The direct and guarded dashboards tracked each other within normal drift. Evidence: `evidence/02-baseline/baseline-readings.md` and the screenshots beside it.

A 10-second capture on the loopback interface, filtered to TCP port 5020, recorded 60 packets (40 Modbus). One pair was analysed in detail: query frame 1 (127.0.0.1:40970 to 127.0.0.1:5020) and response frame 3 (the reverse), both transaction identifier 708, unit 1, function code 3 (Read Holding Registers). The query requested start address 0 and 4 registers; the response returned byte count 8 and the values 669, 366, 985 and 0, which decode through the register map to 66.9 C, 36.6 bar, 98.5 Nm3/h and no alarm. The alarm value is consistent with the pressure, because the plant raises it only at 39.5 bar. The response arrived 1.060 ms after the query. Full field table and raw bytes: `evidence/04-packet-capture/packet-analysis.md`. The plant's own log line for that second is no longer available (the terminal scroll-back does not reach back that far), so the link to process state rests on three checks made from the capture file: the decoded values and alarm bit agree with the register map; two clients polling in the same second received identical values; and across 10 successive responses every second-to-second change stays within the plant's per-tick random-step limits (0.2 C, 0.3 bar, 1.0 Nm3/h). The capture also shows two clients reading the plant once per second each (source ports 40970 and 50874).

This capture shows that a client can read live process state over unauthenticated Modbus/TCP on the direct path and that each packet field maps to a register and a process value; it does not by itself demonstrate the guard's write-side protections, which Section 7 covers separately.

## 5. Local exposure and risk assessment

Full detail in `04-risk-register.csv`. In summary, both the plant (port 5020)
and the guard (port 5021) accept connections from any local client with no
authentication; the guard additionally validates writes to registers 0–2 only,
leaving reads and all other registers unrestricted. All exposure identified
was within the documented loopback scope of the supplied source; no discovery
activity extended beyond `127.0.0.1`. A listening-socket check (`ss -ltnp`) confirmed that the plant (5020), guard (5021) and both dashboards (8080, 8081) listen on 127.0.0.1 only and that none of them requires authentication (`evidence/03-environment/listening-services.txt`).

## 6. Supplied guard analysis

Full detail in `02-guard-analysis.md`, including the complete decision table
and nine source-verified limitations (L-01 through L-09). In summary, the
guard implements three write-side checks (absolute range, single-step limit,
and a 3-write rolling-baseline drift comparison) and a passive safe-mirroring
mechanism, but validates writes only, not reads; validates only three of the
plant's registers; cannot prevent a client from bypassing it by connecting
directly to the plant's own port; and has no anti-replay or rate-limiting
protection. These are described here as implemented facts and documented
limitations, not as a claim that the guard is comprehensive security control.

## 7. Controlled validation and findings

Full scenario-by-scenario detail, including the pre-run predicted behaviour
derived from the source code, is in `03-security-validation-test-plan.md`.

| Scenario | Script | Expected (from source) | Observed | Match? |
|---|---|---|---|---|
| SV-01 | `attack_hold.py` | Direct: unsafe value reaches the plant. Guarded: rejected outright (absolute range) | Direct: 77/77 read-backs unsafe (peak 95.0 bar). Guarded: 0/74; blocked and logged | Yes (direct result stronger than predicted) |
| SV-02 | `false_data_attack.py` | Direct: unsafe values likely reach the plant across the 6 batches. Guarded: rejected outright (absolute range) | Direct: 6/6 unsafe on all three channels. Guarded: 0/6 on all three; all blocked and logged | Yes |
| SV-03 | `slow_drift_attack.py` | Direct: pressure climbs unopposed past the safe ceiling. Guarded: accepted for ~3 steps, then rejected by rolling-drift check from step 4 onward | Direct: actual stayed 39.7 to 40.0 (the plant's own clamp; read-back taken 2 s after each write). Guarded: contained by the absolute-range check from step 2; the drift check never fired alone and first fired at 42.2 bar | No: two parts of the prediction were wrong (see below) |
| SV-04 | `replay_attack.py` | Both paths: replayed in-range value accepted identically all three times (no anti-replay check) | Both runs completed; no BLOCKED event logged for the guarded replays (the complete event log ends at 06:02:03) | Yes (indirect evidence) |
| SV-05 | `recon_scan.py` | Both paths: full register range readable (guard does not validate reads) | Both paths returned registers 0 to 49; guarded view showed alarm = 1 at 39.5 bar | Yes |
| SV-06 | `dos_attack.py` (if approved) | Both paths: all 200 reads complete; no rate limiting in source | Not run | n/a |

Predictions in the table were derived from the source before the run. Two were not borne out for SV-03: the direct path did not show pressure climbing past 40.0 bar, because the script reads back 2 seconds after each write and the plant's own once-per-second clamp had already pulled the value to its 40.0 bar ceiling; and the guarded path was contained by the absolute-range check, because the run started at 38.2 bar, so the drift check was never the deciding control. The complete event log shows the requests at 40.2 and 41.2 bar were blocked by the range check alone, and drift first appeared at 42.2 bar (2.3 bar), by which time the request was already above the ceiling. The source also shows that the guard's sync thread refreshes the drift history every second, so the drift window spans only a few seconds (see `03-security-validation-test-plan.md`, Observed results).

Dashboards were checked after SV-01, SV-02 and SV-03 and showed values inside the safe ranges (guarded PROTECTED, direct SYSTEM NORMAL). SV-04 wrote only in-range values and SV-05 only read, so no separate recovery was needed. Final recovery evidence after all scenarios: both dashboards at 10:53 showed SYSTEM NORMAL and PROTECTED with every value inside its safe range (`evidence/06-recovery/final-dashboards-1053.png`), and after the services were stopped a port check at 11:25 printed "no lab ports listening" (`evidence/06-recovery/ports-closed-after-stop-1125.png`).

## 8. Recommendations

These are proposed improvements, not features already present in the source:

1. Validate writes to every register the plant exposes, not only registers 0–2 (addresses R-02).
2. Add read-side access control or rate limiting to the guard, since reconnaissance is currently as easy through the guard as through the plant directly (addresses R-03).
3. Enforce network policy so only the guard, not the plant directly, is reachable by ordinary clients (addresses R-04).
4. Add a freshness or sequence-number check to the guard's write validation to prevent replay of previously-valid commands (addresses R-05).
5. Add rate limiting/connection throttling to both services (addresses R-06).
6. Give the guarded dashboard an explicit "data stale / plant unsafe via bypass" indicator rather than a silent freeze (addresses R-07).
7. Compare drift against a longer-horizon baseline in addition to the current 3-write rolling window (addresses R-08).

## 9. Recovery and conclusion

All four services (plant, guard and both dashboards) were stopped with Ctrl+C. At 11:25 a check of the listening sockets for ports 5020, 5021, 8080 and 8081 printed "no lab ports listening". Before that, the final dashboards at 10:53 showed SYSTEM NORMAL and PROTECTED with every value inside its safe range. The VM snapshot LAB-START-1 was taken before the lab began, and the evidence files were copied out of the VM before any restore.

This lab demonstrated, using the course's own supplied source rather than a
constructed replacement, that a simple rule-based Modbus/TCP guard can
meaningfully reduce the risk of abrupt or gradual out-of-range process writes
reaching a simulated plant, while still leaving clearly identifiable gaps
(unauthenticated reads, unvalidated non-process registers, no anti-replay
protection, no rate limiting, and a bypassable architecture) that a real OT
security control would need to close. These findings are scoped strictly to
this educational simulation and should not be read as a general claim about
any production system.

## 10. References and appendices

- Course repository: `https://github.com/icdfa/ACICP302ICS-Network-Security-and-Attacks`
- Course source commit used: `2a66f9bcf253eb3fe31c68ad52794cd535421da8`
- Source files cited: `plant_sim/plant.py`, `plant_sim/guard.py`, `plant_sim/dashboard.py`, `plant_sim/dashboard_guarded.py`, `plant_sim/attack_hold.py`, `plant_sim/false_data_attack.py`, `plant_sim/slow_drift_attack.py`, `plant_sim/replay_attack.py`, `plant_sim/recon_scan.py`, `plant_sim/dos_attack.py`
- Course reference material cited: `docs/reference-architecture.md`
- Appendices: `01-topology-and-register-map.md`, `02-guard-analysis.md`, `03-security-validation-test-plan.md`, `04-risk-register.csv`, `05-evidence-log.md`, `evidence/` (screenshots, PCAP, `events.jsonl` extracts)

**Statement of attribution:** the simulated plant, guard, dashboards and
demonstration scripts referenced throughout this report are course-supplied
source code from `labs/ot-security/student-lab-source/`, not original
implementation work by the learner. The analysis, evidence, test execution
and conclusions in this report are the learner's own.
