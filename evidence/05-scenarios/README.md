# Scenario Evidence

Run on 2026-10-15 (VM clock, which runs on West African Time), one scenario at a time, supplied scripts with unchanged parameters, direct path
(5020) then guarded path (5021). Each folder holds the verbatim console output, screenshots where taken, and that run's guard events.

`events-full.jsonl` is the complete guard event log copied from `plant_sim/events.jsonl` (103 events, 05:52:54 to 06:02:03).
Event counts reconcile with the console output: 74 events for the 74 attack_hold writes, 18 events for the 6 x 3 false_data writes,
and 11 events for slow_drift steps 2 to 12 (step 1 was accepted, so it logged nothing).

| Folder | Script | Run times (VM) | Guard events |
|---|---|---|---|
| SV-01-attack_hold | attack_hold.py | about 05:51 (5020), 05:52:54 to 05:53:02 (5021) | 74 |
| SV-02-false_data_attack | false_data_attack.py | about 05:57 (5020), 05:57:48 to 05:57:50 (5021) | 18 |
| SV-03-slow_drift_attack | slow_drift_attack.py | about 06:01 (5020), 06:01:41 to 06:02:03 (5021) | 11 |
| SV-04-replay_attack | replay_attack.py | after 06:03 (exact time not recorded) | 0 |
| SV-05-recon_scan | recon_scan.py | after SV-04 (exact time not recorded) | n/a (reads are not logged) |
| SV-06 dos_attack.py | not run (needs separate instructor approval) | | |
