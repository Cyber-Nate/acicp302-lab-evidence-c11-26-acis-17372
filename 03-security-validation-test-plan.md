# Controlled Validation Test Plan - Supplied Simulation

**Learner:** Nathaniel Temitope Ogunjobi
**Date and WAT time:** 2026-10-15, VM clock 05:50 to about 06:05 (the VM clock runs on West African Time and agrees with the corrected Windows clock)
**Assigned Ubuntu VM/snapshot:** Ubuntu 64-bit guest on VMware Workstation (hostname cyber-nate-VMware-Virtual-Platform); snapshot: LAB-START-1
**Course source commit:** `2a66f9bcf253eb3fe31c68ad52794cd535421da8`
**Canonical source path:** `labs/ot-security/student-lab-source/`
**Authorized target:** `127.0.0.1` only
**Instructor approval reference, if required:** not applicable; SV-06 (`dos_attack.py`) was not run

Every scenario below uses a supplied script with its default, unmodified
arguments, run one at a time, first against the direct path (port 5020) and
then the guarded path (port 5021), per the Student Lab Guide. Five scenarios
are planned, exceeding the brief's 3-scenario minimum. "Expected behaviour" is
derived directly from reading `plant.py` and `guard.py` (marked **predicted
from source**); it is a prediction to be checked against your actual run, not
a guaranteed result - timing-dependent variation is expected and should be
reported honestly if it occurs.

| Scenario ID | Supplied script and local path | Starting state | Expected process/guard behaviour (predicted from source) | Evidence to collect | Stop condition | Recovery action | Result |
|---|---|---|---|---|---|---|---|
| SV-01 | `attack_hold.py 5020` then `attack_hold.py 5021` | Pressure at normal baseline (~34.0 bar) on both paths | **Direct (5020):** plant has no write validation; the forced value (95.0 bar) is readable immediately after each write, so most of the ~80 read-backs over 8s should show the unsafe value before the plant's 1s self-heal tick clamps it back toward the 40.0 bar ceiling - script should print "UNSAFE VALUE REACHED THE PLANT". **Guarded (5021):** 95.0 bar is far outside the absolute range (30.0–40.0 bar) and is rejected on the first write; guard's exposed pressure never moves - script should print "attack fully contained" | Script console output (writes sent, peak pressure, unsafe-reading fraction); dashboard screenshots during the run; relevant `events.jsonl` lines for the guarded run | 8-second script timeout (built in); Ctrl+C if VM becomes unstable | Allow plant's 1s tick to self-heal pressure back into range; confirm both dashboards show normal baseline again before the next scenario | Run 2026-10-15; see Observed results |
| SV-02 | `false_data_attack.py 5020` then `false_data_attack.py 5021` | All three values at normal baseline on both paths | **Direct:** all three written values (110.0 °C, 55.0 bar, 200.0 Nm³/h) are far outside their ranges; some read-backs across the 6 batches are likely to catch unsafe values before each tick corrects them. **Guarded:** all three writes in each batch are rejected independently by the absolute-range check (none are close to their limits); guard state should remain unchanged across all 6 batches | Script console output (peak values per channel, unsafe-reading counts per channel); three `events.jsonl` lines per rejected batch (temp/pressure/flow each log separately) | 6-iteration script completion (~6s) | Confirm plant self-heals to baseline on the direct run | Run 2026-10-15; see Observed results |
| SV-03 | `slow_drift_attack.py 5020` then `slow_drift_attack.py 5021` | Pressure at normal baseline | **Direct:** no step or drift check exists on the plant itself; 12 unopposed +1.0 bar steps should carry pressure roughly 12.0 bar above its starting point, i.e. past the 40.0 bar safe ceiling by the later steps. **Guarded:** each individual +1.0 bar step is well under the 5.0 bar step limit, so the step check alone will not catch this. However, the rolling-baseline drift check compares each request to the value accepted 3 writes ago (threshold 2.0 bar): writes 1–3 should be accepted (building the history), and from write 4 onward the cumulative 3-step drift (3 × 1.0 bar = 3.0 bar) exceeds the 2.0 bar threshold, so write 4 and essentially every write after it should be rejected, holding the guarded pressure near its value after step 3 (roughly 3.0 bar above baseline) for the remainder of the run. This scenario is the clearest demonstration of why the drift check exists: the step check alone would miss it | Full step-by-step script output (`requested` vs `actual` for all 12 steps) on both paths; this is the single most important comparison for the guard-analysis section, since the guarded output should visibly plateau partway through while the direct output keeps climbing | 12-step script completion (~24s) | Confirm plant drifts back toward baseline afterward; verify guard's drift history naturally clears after `DRIFT_WINDOW` normal accepted syncs | Run 2026-10-15; see Observed results |
| SV-04 | `replay_attack.py 5020` then `replay_attack.py 5021` | Pressure at normal baseline | The replayed value itself (+0.5 bar) is well within range/step/drift limits on both paths, so it is expected to be **accepted identically all three times on both the direct and guarded paths**. This is the intended finding: none of the guard's three checks are sequence- or identity-aware, so a previously-valid write can be resent and re-accepted without detection, demonstrating the absence of anti-replay protection (limitation L-05) rather than a blocked attack | Script console output confirming all three writes went through; dashboard readings before/after | Script completion (~1s) | None required; value stays in safe range throughout | Run 2026-10-15; see Observed results |
| SV-05 | `recon_scan.py 5020` then `recon_scan.py 5021` | Any state | `GuardedContext` only overrides `setValues` (writes), never `getValues` (reads), so reads against the guard pass straight to its local store unmodified, identically to reading the plant directly. **Both runs are expected to successfully enumerate registers 0–49** (showing live process values in 0–3 and zero-padding in 4–49), demonstrating the guard provides no read-side access control at all | Script console output (all five register blocks, both paths) | Script completion (~0.5s) | None required; read-only | Run 2026-10-15; see Observed results |
| SV-06 (requires separate instructor approval) | `dos_attack.py 5020` then `dos_attack.py 5021` | Normal baseline, services idle otherwise | Neither `plant.py` nor `guard.py` implements rate limiting or connection throttling, so the expected baseline behaviour is that both servers complete all 200 reads (10 threads × 20 reads) without crashing, though response latency may increase under concurrent load. This demonstration tests availability/resilience rather than data integrity; do not run without the instructor approval referenced above | Script console output ("Done. 200 requests sent."); dashboard responsiveness observation during the run; note any errors or dropped connections | Script completion | Confirm both dashboards are still responsive afterward | Not run |

## Comparison summary

| Scenario | Baseline state | Observed direct/guarded state | Log/packet/dashboard reference | Finding/limitation | Recovery verified? |
|---|---|---|---|---|---|
| SV-01 | Direct dashboard 05:50: 66.1 C / 39.2 bar / 67.3 Nm3/h, SYSTEM NORMAL | Direct: 77/77 read-backs unsafe (peak 95.0 bar). Guarded: 0/74 unsafe | `evidence/05-scenarios/SV-01-attack_hold/` console files, guarded dashboard 05:53, `events-5021.jsonl` | Unprotected plant accepts any write magnitude; the guard drops the write (range, step and drift all listed in one reason) | Yes: guarded dashboard in range and PROTECTED at 05:53; direct SYSTEM NORMAL at 06:01 |
| SV-02 | No pre-run screenshot taken; both views in range | Direct: 6/6 unsafe on temperature, pressure and flow. Guarded: 0/6 on all three | `SV-02-false_data_attack/` console files, guarded dashboard 05:58, `events-5021.jsonl` | Same pattern across all three process variables at once | Yes: guarded dashboard 05:58 in range (74.1 C / 39.7 bar / 144.1 Nm3/h); direct SYSTEM NORMAL at 06:01 |
| SV-03 | Direct start 39.3 bar; guarded start 38.2 bar | Direct: actual stayed 39.7 to 40.0 at every step. Guarded: actual stayed 39.3 to 40.0; every logged block also failed the range check | `SV-03-slow_drift_attack/` console files, guarded dashboard 06:03, `events-5021.jsonl` | Contained by the absolute-range check, not isolated to the drift check; the plant's own 1-second clamp explains the direct result (see Observed results) | Yes: guarded dashboard 06:03 in range (74.2 C / 38.0 bar / 140.4 Nm3/h) |
| SV-04 | Values in range (replayed values 36.9 and 35.9 bar are inside 30.0 to 40.0) | Both runs completed; no BLOCKED event after the guarded replays | `SV-04-replay_attack/` console files; the complete event log ends at 06:02:03 | L-05: no anti-replay or freshness check (indirect evidence for the guarded path) | Not needed (writes stayed in range) |
| SV-05 | Any state | Both paths returned registers 0 to 49 (guarded view showed alarm = 1 at 39.5 bar) | `SV-05-recon_scan/` console files | L-02: the guard validates writes only, not reads | Not needed (read-only) |
| SV-06 | n/a | Not run | n/a | L-06 stays source-derived only | n/a |

## Observed results (run on 2026-10-15, VM clock 05:50 to about 06:05)

All five scenarios were run one at a time with the supplied scripts and default parameters, direct path first and then guarded path.
Full output is in `evidence/05-scenarios/`.

| Scenario | Direct path (5020) | Guarded path (5021) | Guard event evidence | Matches the prediction? |
|---|---|---|---|---|
| SV-01 attack_hold | 77 writes; peak 95.0 bar at t=0.09 s; 77 of 77 read-backs (100%) outside the safe range; "UNSAFE VALUE REACHED THE PLANT" | 74 writes; peak 38.2 bar (the plant's own value, not the attack value); 0 of 74 outside the range; "attack fully contained" | All 74 writes blocked (74 events, 05:52:54 to 05:53:02), each like "95.0 bar outside safe range 30.0-40.0 bar; step of 57.5 bar exceeds max 5.0 bar; cumulative drift of 57.7 bar over last 3 accepted writes exceeds 2.0 bar limit" | Yes. The direct result was stronger than predicted (100%, not "most"). |
| SV-02 false_data | Peaks 110.0 C, 55.0 bar and 200.0 Nm3/h at t=0.01 s; unsafe readings 6/6 on all three channels; "UNSAFE VALUES REACHED THE PLANT" | Peaks 74.5 C, 39.5 bar and 144.3 Nm3/h (the plant's own in-range values); unsafe 0/6 on all channels; "attack fully contained" | 18 events at 05:57:48 to 05:57:50 (6 batches x 3 channels, so every write was blocked), each listing range, step and drift | Yes |
| SV-03 slow_drift | Start 39.3 bar; requested 40.3 up to 51.3; actual stayed between 39.7 and 40.0 at every step | Start 38.2 bar; requested 39.2 up to 50.2; actual stayed between 39.3 and 40.0 at every step | 11 events from 06:01:43 (requested 40.2) to 06:02:03 (50.2); step 1 (39.2 bar) was accepted and logged nothing. Every event lists the range check; drift first appears at 42.2 bar (2.3 bar); step first appears at 45.2 bar | Partly. See the prediction check below. |
| SV-04 replay | "Replayed write 36.9 bar three times." No verdict printed | "Replayed write 35.9 bar three times." No verdict printed | No BLOCKED line after the replays: the complete event log ends at 06:02:03 (SV-03's last step) | Yes (indirect evidence for the guarded path) |
| SV-05 recon | Registers 0 to 9 = [725, 382, 1323, 0, ...]; registers 10 to 49 all 0 | Registers 0 to 9 = [723, 395, 1340, 1, ...]; registers 10 to 49 all 0 | none (reads are neither validated nor logged) | Yes |
| SV-06 dos | Not run | Not run | n/a | n/a |

### SV-03 prediction check (parts of the pre-run prediction were wrong)

1. **Direct path.** I predicted pressure would climb about 12 bar past the 40.0 bar ceiling. It did not: `actual` never exceeded 40.0 bar.
   The script reads the register 2 seconds after each write (`slow_drift_attack.py`), and the plant's own 1-second loop
   (`simulate_normal_operation` in `plant.py`) clamps pressure back to its 40.0 bar range ceiling on every tick, so by the read-back the
   value had already been pulled down to 39.7 to 40.0 bar (the ceiling plus a small random jitter). The plant still accepted every write
   without validation, which SV-01 shows directly (77 of 77 unsafe read-backs when reading immediately after each write). SV-03 therefore
   shows requested and actual values diverging because of the plant's own clamp, not sustained unsafe pressure.
2. **Guarded path.** I predicted three accepted steps followed by drift-check rejections from step 4. The complete event log
   (`evidence/05-scenarios/events-full.jsonl`) shows what happened instead. The run started at 38.2 bar, so step 1 (39.2 bar) was accepted and
   logged nothing, and steps 2 to 12 (40.2 to 50.2 bar, 11 events) were all rejected. The requests at 40.2 and 41.2 bar were rejected by the
   absolute-range check alone: the drift check did not fire for them (their drift from the baseline was under the 2.0 bar threshold, roughly 0.3 and
   1.3 bar, extrapolating back from the 2.3 bar logged at 42.2 bar). The drift check first fired at 42.2 bar, when the request was already 2.2 bar above the
   safe ceiling, and from 45.2 bar the step check fired as well. So the ramp was contained by the absolute-range check, and the drift check lagged
   behind it and never acted alone.
3. **Source correction.** `sync_safe_reads` in `guard.py` also appends each safe mirrored plant reading to the same accepted-history deque
   every second. The drift baseline therefore refreshes continuously and spans only the last three entries (a few seconds), not the last three
   attacker writes. A ramp of 1.0 bar every 2 seconds (about 1.5 bar across three entries) stays under the 2.0 bar threshold, so the drift check
   would not be expected to stop it before the absolute limit. This is derived from the source and was not demonstrated; a run starting well
   below 40 bar would test it, and it was not run.

### Other observations

- The process sat near the top of its range during the session (pressure 38.0 to 39.7 bar, flow up to 144.4 Nm3/h on the dashboards). The plant's
  high-pressure alarm (raised at 39.5 bar and above) was active on the guarded recon read (register 3 = 1 at pressure 39.5 bar).
- `_check` in `guard.py` collects every violated check and `setValues` joins them with "; ", so one blocked write usually lists range,
  step and drift together.

## Completion confirmation

I confirm that every scenario that was run (SV-01 to SV-05; SV-06 was not run) was run only against the documented loopback
simulation in the assigned VM, using supplied parameters unchanged, and that the four services were stopped afterward and the
loopback ports were confirmed closed.

**Learner signature:** Nathaniel Temitope Ogunjobi
**Date:** 15 October 2026
