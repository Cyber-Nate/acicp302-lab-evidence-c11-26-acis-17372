# Evidence Index

Every file below was created by the learner during the lab run. Times are VM time unless stated.

| Path | What it shows | Supports |
|---|---|---|
| `01-preflight/preflight-loopback-check.txt` | Source commit, both loopback grep checks and their output | Source version and scope/preflight record |
| `01-preflight/pip-install-output.txt` | Virtual environment and pinned dependency installation | Preflight record |
| `02-baseline/baseline-1-guarded-dashboard-8081.png` | Guarded dashboard, PROTECTED, 68.0 C / 31.5 bar / 95.0 Nm3/h | Normal baseline |
| `02-baseline/baseline-1-direct-dashboard-8080.png` | Direct dashboard, SYSTEM NORMAL, 69.9 C / 30.9 bar / 97.9 Nm3/h | Normal baseline |
| `02-baseline/baseline-1-service-terminals.png` | Plant telemetry, guard and guarded-dashboard startup lines | Normal baseline, service startup |
| `02-baseline/baseline-2-direct-dashboard-8080.png` | Direct dashboard, 72.4 C / 30.4 bar / 101.8 Nm3/h | Normal baseline |
| `02-baseline/baseline-2-guarded-dashboard-8081.png` | Guarded dashboard, 71.1 C / 30.1 bar / 101.6 Nm3/h | Normal baseline |
| `02-baseline/baseline-readings.md` | Tabulated readings, clock note, service interruption record | Normal baseline |
| `03-environment/wireshark-install-and-network-state.png` | Network adapter check and Wireshark installation | Scope record (adapter state) |
| `03-environment/network-and-tool-check.txt` | Adapter state before and after installation | Scope record |
| `04-packet-capture/capture-commands.txt` | Exact capture commands, the failed first attempt, the 60-packet result | Packet capture |
| `04-packet-capture/dumpcap-capture-terminal.png` | Terminal showing the capture result and Wireshark launch | Packet capture |
| `04-packet-capture/modbus-filter-packet-list-a.png`, `-b.png` | Filtered Modbus packet list (40 of 60 packets) | Request/response analysis |
| `04-packet-capture/query-frame1-trans708.png` | Query packet, transaction 708, fields expanded | Request/response analysis |
| `04-packet-capture/response-frame3-trans708.png` | Matching response, registers 669 / 366 / 985 / 0 | Request/response analysis |
| `04-packet-capture/packet-analysis.md` | Field-by-field analysis tied to the register map | Request/response analysis |
| `04-packet-capture/normal-baseline-lo-5020.pcapng` | The learner-created loopback capture, 60 packets (SHA-256 `deff65b74a12a662a689ccd69bb4d48357d8f031e79311b3ab42afeda5784a2a`) | Learner-created PCAP |
| `05-scenarios/SV-01` to `SV-05` | Verbatim console output, dashboard screenshots, guard event lines for each scenario (see `05-scenarios/README.md`) | Test results, guard decisions |
| `05-scenarios/events-full.jsonl` | The complete guard event log, 103 events (SHA-256 `8391fd79318ddf5472bd27ba8a9d6fceafd0cbfed10e60520eeda0eedf07fd8a`) | Guard decision evidence |
| `06-recovery/final-dashboards-1053.png` | Both dashboards after all scenarios: SYSTEM NORMAL and PROTECTED, values in range | Cleanup and recovery |
| `06-recovery/ports-closed-after-stop-1125.png` | Port check after stopping the services: `no lab ports listening` | Cleanup and recovery |
| `03-environment/listening-services.txt`, `listening-services-ss-grep.png` | `ss -ltnp` showing the four lab services bound to 127.0.0.1 only | Local service inventory |
