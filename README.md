# ACICP302ICS - Module 1 Practical Lab Evidence

**Course:** ACICP302ICS - ICS Network Security & Attacks
**Activity:** Analyze, Test & Defend the Supplied Modbus/TCP OT Simulation
**Learner:** Nathaniel Temitope Ogunjobi
**Registration number:** C11/26/ACIS/17372
**Cohort:** ACIS-COHORT-11, Level 1, Semester 1
**Submission deadline:** 15 Oct 2026, 11:59 PM WAT

## Attribution

This repository contains my own analysis, evidence and report for Practical Lab
Activity "Analyze, Test & Defend the Supplied Modbus/TCP OT Simulation." The
simulated plant, guard and demonstration scripts in `plant_sim/` are course-
supplied source, **not my original implementation**. They are used here only
as the subject of analysis, exactly as the lab brief requires.

- Course repository: https://github.com/icdfa/ACICP302ICS-Network-Security-and-Attacks
- Canonical student source used: `labs/ot-security/student-lab-source/`
- Course source commit used: `2a66f9bcf253eb3fe31c68ad52794cd535421da8`
- Assigned Ubuntu VM / snapshot: Ubuntu 64-bit guest on VMware Workstation (hostname cyber-nate-VMware-Virtual-Platform); snapshot name: LAB-START-1
- Authorized scope: loopback (`127.0.0.1`) only, inside the assigned Ubuntu VM. No bridged
  networking, no port forwarding, no real or external systems.

## Repository contents

| File | Contents | Maps to brief evidence item |
|---|---|---|
| `01-topology-and-register-map.md` | Annotated architecture, component inventory, verified Modbus register map | Evidence item 1, 2 |
| `02-guard-analysis.md` | `guard.py` decision table, event-log evidence, assumptions and limitations | Evidence item 4 |
| `03-security-validation-test-plan.md` | Approved test plan, 5 scenarios (exceeds the 3-scenario minimum), expected-vs-observed comparison | Evidence item 5 |
| `04-risk-register.csv` | Local service/exposure risk register (course template format) | Evidence item 3 |
| `05-evidence-log.md` | Chronological evidence log and safety declaration (course template format) | Evidence item 1, supports all |
| `06-final-report.md` | Full final report (course template format) | Evidence item 6 |
| `evidence/` | Screenshots, text logs, PCAP/PCAPNG, `events.jsonl` extracts; start with `evidence/README.md` (index) | Evidence items 2, 3, 5 and the preflight record |

## Status

Complete and documented in `evidence/`: preflight and loopback check, dependency installation, service startup, two baseline dashboard
pairs, adapter-state record, local service inventory, a learner-created loopback capture (`normal-baseline-lo-5020.pcapng`) with
request/response analysis, five demonstration scenarios (SV-01 to SV-05) on both the direct and guarded paths with console output, dashboard
screenshots and the complete guard event log, and recovery evidence (final dashboards and a port check after stopping the services).

Before submitting: replace the one remaining `[INSERT]` in `06-final-report.md` (your private repository URL and final commit hash).

Predicted behaviour derived from reading the source code is marked **(predicted from source)**. Where a prediction did not hold (SV-03), the
report records what actually happened and why, exactly as the Student Lab Guide requires.

## Safety declaration

I confirm this evidence will be gathered only in the assigned ICDFA isolated
Ubuntu simulation using the canonical student source at
`labs/ot-security/student-lab-source/`, that no real or external system will be
targeted, that `labs/ot-security/archive-original/` will not be executed, and
that the course-provided source is attributed above and not represented as my
own code.

**Signature:** Nathaniel Temitope Ogunjobi
**Date:** 15 October 2026
