The script prints no accept/reject verdict. The complete guard event log (../events-full.jsonl, 103 events) ends at 2026-10-15 06:02:03,
which is the last step of SV-03. The SV-04 replays ran after that and produced no BLOCKED event, so the guard raised no objection to the
repeated identical in-range writes. This is indirect evidence that they were accepted (the guard logs only rejections).
