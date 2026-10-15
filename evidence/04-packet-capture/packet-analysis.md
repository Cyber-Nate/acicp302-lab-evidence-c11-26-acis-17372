# Normal-Operation Modbus/TCP Packet Analysis

## Capture details

| Item | Value |
|---|---|
| Interface | `lo` only (loopback) |
| Capture filter | `tcp port 5020` (plant Modbus/TCP port) |
| Duration | 10 seconds (`-a duration:10`) |
| Packets captured | 60 (60 received, 0 dropped) |
| Modbus packets after decoding | 40 (filter `modbus`); the other 20 are pure TCP acknowledgements (ACK flag only, no payload), confirmed by parsing the capture file |
| First packet | 2026-10-14 17:06:37.738610515 on the VM clock (16:06:37 UTC, so the VM runs on West African Time, UTC+1); the 60 packets span 9.01 s |
| Decode setting | Analyze > Decode As > TCP port 5020 = Modbus/TCP (Wireshark only auto-decodes port 502) |
| Capture file | `normal-baseline-lo-5020.pcapng` in this folder (6,920 bytes, SHA-256 `deff65b74a12a662a689ccd69bb4d48357d8f031e79311b3ab42afeda5784a2a`) |
| Scope | Every frame is 127.0.0.1 to 127.0.0.1 inside the assigned VM. The ethernet adapter was disconnected. |

Screenshots: `modbus-filter-packet-list-a.png`, `modbus-filter-packet-list-b.png` (full filtered list),
`query-frame1-trans708.png`, `response-frame3-trans708.png` (the selected pair).

## Selected request and matching response (transaction 708)

Isolated with the display filter `mbtcp.trans_id == 708`, which returns exactly two packets.

| Field | Query (frame 1) | Response (frame 3) |
|---|---|---|
| Time (VM) | 17:06:37.738610515 | 17:06:37.739670691 |
| Source to destination | 127.0.0.1:40970 to 127.0.0.1:5020 | 127.0.0.1:5020 to 127.0.0.1:40970 |
| Frame length | 78 bytes | 83 bytes |
| TCP | Seq 1, Ack 1, Len 12 | Seq 1, Ack 13, Len 17 |
| Modbus/TCP transaction identifier | 708 | 708 |
| Protocol identifier | 0 | 0 |
| MBAP length | 6 | 11 |
| Unit identifier | 1 | 1 |
| Function code | 3 (Read Holding Registers) | 3 (Read Holding Registers) |
| Exception | No | No |
| Reference number (start address) | 0 | not present |
| Word count / byte count | Word count 4 | Byte count 8 |
| Time from request | not applicable | 1.060176 ms |

Raw Modbus/TCP bytes (from the packet bytes pane):

- Query: `02 c4 00 00 00 06 01 03 00 00 00 04`
  (transaction 0x02c4 = 708, protocol 0, length 6, unit 1, function 3, start address 0, quantity 4)
- Response: `02 c4 00 00 00 0b 01 03 08 02 9d 01 6e 03 d9 00 00`
  (same transaction 708, length 11 = unit + function + byte count + 8 data bytes, byte count 8, then four 16-bit registers)

The response arrived 1.060 ms after the query, which matches the difference between the two packet
timestamps (17:06:37.739670691 minus 17:06:37.738610515). The response echoes the query's transaction
identifier, which is how a client matches a response to its request.

## Register values returned and what they mean

Register meanings, scales and safe ranges come from `01-topology-and-register-map.md`
(taken from the supplied `plant.py`).

| Register | Raw value | Hex | Point | Scale | Value | Safe range | Result |
|---:|---:|---|---|---|---:|---|---|
| 0 | 669 | 0x029d | Temperature | tenths of C | 66.9 C | 60.0-75.0 C | inside range |
| 1 | 366 | 0x016e | Pressure | tenths of bar | 36.6 bar | 30.0-40.0 bar | inside range |
| 2 | 985 | 0x03d9 | Flow | tenths of Nm3/h | 98.5 Nm3/h | 50.0-150.0 Nm3/h | inside range |
| 3 | 0 | 0x0000 | High-pressure alarm | 0 or 1 | 0 (no alarm) | alarm set at 39.5 bar or above | consistent: 36.6 bar is below 39.5 bar |

The alarm register agreeing with the pressure register is an internal consistency check on the
decoding: the plant sets register 3 to 1 only when pressure reaches 39.5 bar (raw 395).

## Relating the packet to process state

The plant's own telemetry log line for 17:06:37 is no longer available: the terminal scroll-back does not reach back that far after
about 18 hours of once-per-second logging. The link between packet and process state therefore rests on three checks that can be made
from the capture file itself, which the analysis below verified by parsing it:

1. **Values and alarm agree with the register map.** The decoded values are inside their safe ranges, and the alarm register (0) agrees
   with the pressure (36.6 bar is below the 39.5 bar alarm threshold).
2. **Both clients saw the same process state.** The two clients that polled in the same second received identical register values in all
   10 polls, as expected when both read one plant.
3. **Successive values follow the plant's own drift model.** Across the 10 successive responses to the client on port 40970 (table below), every
   second-to-second change stays within the plant's per-tick random-step limits in `plant.py` (temperature at most 0.2 C, pressure at most 0.3 bar,
   flow at most 1.0 Nm3/h), and the alarm stays 0 while pressure stays under 39.5 bar.

| Seconds from first packet | Temperature | Pressure | Flow | Alarm |
|---:|---:|---:|---:|---:|
| 0.0 | 66.9 C (669) | 36.6 bar (366) | 98.5 Nm3/h (985) | 0 |
| 1.0 | 66.9 C (669) | 36.4 bar (364) | 98.1 Nm3/h (981) | 0 |
| 2.0 | 66.7 C (667) | 36.2 bar (362) | 99.0 Nm3/h (990) | 0 |
| 3.0 | 66.6 C (666) | 36.1 bar (361) | 99.1 Nm3/h (991) | 0 |
| 4.0 | 66.6 C (666) | 36.4 bar (364) | 100.0 Nm3/h (1000) | 0 |
| 5.0 | 66.6 C (666) | 36.6 bar (366) | 100.9 Nm3/h (1009) | 0 |
| 6.0 | 66.6 C (666) | 36.5 bar (365) | 101.1 Nm3/h (1011) | 0 |
| 7.0 | 66.8 C (668) | 36.3 bar (363) | 101.9 Nm3/h (1019) | 0 |
| 8.0 | 66.8 C (668) | 36.6 bar (366) | 102.9 Nm3/h (1029) | 0 |
| 9.0 | 66.9 C (669) | 36.6 bar (366) | 102.2 Nm3/h (1022) | 0 |

The dashboard readings in `../02-baseline/baseline-readings.md` (taken at other times) fall in the same ranges, but they are not simultaneous with
this capture and are not used as proof of correlation.

## Observations

1. **Two clients read the plant.** The 40 Modbus packets in 10 seconds are two clients polling once per second
   (2 clients x 10 polls x query and response). The transaction identifiers form two interleaved sequences
   (693 to 697 and 708 to 712 in the first five seconds), and the trans 708 sequence comes from source
   port 40970. From the supplied source, the two clients on port 5020 are the direct dashboard
   (`dashboard.py`, which reads holding registers 0 to 3 whenever the browser polls `/api/state`) and the guard's
   background sync thread (`guard.py`, which polls the plant once per second). The second sequence (transactions 693 to 702) comes from source port 50874, and the trans 708 sequence (708 to 717) from port 40970. The capture does not show which port belongs to which program; both clients received identical values in every second.
2. **The guarded dashboard does not appear on port 5020.** It reads from the guard on port 5021, so it is outside this capture.
3. **No authentication or encryption.** The register values are readable directly in the packet bytes. Any local
   process able to connect to 127.0.0.1:5020 can send the same query, and the plant gives no indication of who asked.
   This is the evidence for risk R-01 and R-03 in `04-risk-register.csv`.
4. **Normal traffic shape.** Short, fixed-size queries (12 bytes of Modbus/TCP payload) and responses (17 bytes),
   response times of about 0.2 to 1.1 ms across the 20 pairs (0.23 to 1.06 ms), no exceptions. This is the baseline against which the demonstration traffic can be compared.

## What this capture shows and does not show

It shows that a legitimate client can read live process state over unauthenticated Modbus/TCP on the direct path,
and it ties each packet field to a register in the data model and to the dashboard process state. It does not
demonstrate the guard's write-side protections, which are tested separately in
`../../03-security-validation-test-plan.md`.
