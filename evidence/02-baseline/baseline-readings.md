# Baseline Readings (before any demonstration)

All values were read from the learner's own screenshots in this folder. Two clocks appear in the screenshots: the VM clock (Ubuntu top bar and the service logs) and the Windows host clock in the taskbar.
The Windows clock was wrong earlier in the session (it ran up to about 8 hours behind after a hard-disk change) and was later corrected to
West African Time, so early host-clock readings (for example 12:34 AM beside a VM clock of 16:30) are unreliable and should be disregarded.
After the correction the two clocks agree (05:50 on both, 10:53 on both, 11:25 on both). The capture file's timestamps (16:06:37 UTC) correspond to
17:06:37 on the VM clock, so the VM runs on West African Time (UTC+1). All timestamps in this repository are VM time.

Safe ranges from the supplied source: temperature 60.0-75.0 C, pressure 30.0-40.0 bar,
flow 50.0-150.0 Nm3/h.

## Dashboard readings

| # | Dashboard | VM time | Host clock (as shown) | Temperature | Pressure | Flow | Banner | Screenshot |
|---|---|---|---|---:|---:|---:|---|---|
| 1 | Guarded, :8081 | 16:30 | 12:34 AM | 68.0 C | 31.5 bar | 95.0 Nm3/h | PROTECTED; guard log "Waiting for attacks..." | baseline-1-guarded-dashboard-8081.png |
| 2 | Direct, :8080 | 16:31:50 ("Live" stamp) | 12:35 AM | 69.9 C | 30.9 bar | 97.9 Nm3/h | SYSTEM NORMAL | baseline-1-direct-dashboard-8080.png |
| 3 | Direct, :8080 | 16:39:07 ("Live" stamp) | 8:43 AM | 72.4 C | 30.4 bar | 101.8 Nm3/h | SYSTEM NORMAL | baseline-2-direct-dashboard-8080.png |
| 4 | Guarded, :8081 | 16:39 | 8:44 AM | 71.1 C | 30.1 bar | 101.6 Nm3/h | PROTECTED; guard log "Waiting for attacks..." | baseline-2-guarded-dashboard-8081.png |

## Plant telemetry log lines (terminal 1, `plant.py`)

| VM time | Temperature | Pressure | Flow | Alarm | Source screenshot |
|---|---:|---:|---:|---:|---|
| 16:32:38 | 70.4 C | 30.7 bar | 95.2 Nm3/h | 0 | baseline-1-service-terminals.png |
| 16:32:45 | 70.4 C | 30.5 bar | 97.1 Nm3/h | 0 | baseline-1-service-terminals.png |
| 17:00:47 | 65.3 C | 32.2 bar | 81.5 Nm3/h | (not visible) | ../03-environment/wireshark-install-and-network-state.png |
| 17:00:48 | 65.4 C | 32.4 bar | 82.1 Nm3/h | (not visible) | ../03-environment/wireshark-install-and-network-state.png |

## Normal-operation observations

- All readings sit inside the safe ranges and no alarm was active (plant `alarm=0`).
- The supplied guide quotes approximately 67.0 C, 34.0 bar and 90.0 Nm3/h at start. The plant drifts by small random steps each second, so the observed values differ: temperature 65-72 C, pressure 30-33 bar, flow 81-102 Nm3/h across the session. Pressure stayed close to the 30.0 bar lower bound in these samples. By 17:06:37 the captured Modbus response (see ../04-packet-capture/packet-analysis.md) showed 66.9 C, 36.6 bar and 98.5 Nm3/h, so the process wanders widely within range over tens of minutes.
- The direct and guarded dashboards agree to within normal drift (differences of about 1-2 units in these pairs). Each pair was taken about a minute apart, so the differences reflect drift over time and not a disagreement between the two views.
- The guarded dashboard shows the "PROTECTED" banner and an empty guard detection log, as expected before any demonstration.

## Service startup record

- Guard (terminal 2): `2026-10-14 16:25:12,800 INFO Guard listening on 127.0.0.1:5021 - protecting plant at 127.0.0.1:5020`, then `16:25:12,804 INFO Server listening.`
- Guarded dashboard (terminal 4): `Running on http://127.0.0.1:8081`.
- Plant (terminal 1): telemetry log lines once per second; the plant startup line `...listening on 127.0.0.1:5020` is only partly visible in baseline-1-service-terminals.png. The startup line itself was not captured; the plant's operation is evidenced by its telemetry lines, the plant listening on 127.0.0.1:5020 (`../03-environment/listening-services.txt`) and the guard connecting to it.
- Direct dashboard (terminal 3): served `http://127.0.0.1:8080`.

## Service interruption (record exactly what happened)

The guard terminal logged `ERROR Sync error: Modbus Error: [Connection] Failed to connect[ModbusTcpClient 127.0.0.1:5020]` and `Connection to (127.0.0.1, 5020) failed: [Errno 111] Connection refused` at about 16:48:30 VM time (visible in ../03-environment/wireshark-install-and-network-state.png). This is the guard's sync thread failing while the plant was not listening. The plant was running again by 17:00 (telemetry visible). Per the learner, all four services were stopped and restarted around the Wireshark installation; exact times were not recorded.
