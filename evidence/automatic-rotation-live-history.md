# Automatic three-unit rotation from live Home Assistant history

This record is derived from Home Assistant recorder history for the live shelf.
Unit labels are anonymized and correspond to the same 2013, 2014, and 2019 units
used elsewhere in this repository. Times below are local EDT (UTC−04:00).

## Result

Ten consecutive scheduled handovers completed at the configured 24-hour cadence.
Each followed the same make-before-break sequence:

1. one parked unit was released;
2. the active count changed from one to two;
3. both units ran for exactly 60 seconds; and
4. the outgoing unit parked, returning the active count to one.

There were no zero-active events. Across the two minutes before and after every
handover, a rectifier's reported output voltage remained between 53.91 and
54.10 V.

| Local start | Outgoing | Incoming | Two-active overlap | Output voltage window |
|---|---|---|---:|---:|
| 2026-09-21 21:15 | Unit C | Unit B | 60 s | 53.93–54.08 V |
| 2026-09-22 21:15 | Unit B | Unit C | 60 s | 53.95–54.10 V |
| 2026-09-23 21:15 | Unit C | Unit A | 60 s | 53.93–54.08 V |
| 2026-09-24 21:15 | Unit A | Unit C | 60 s | 53.91–54.08 V |
| 2026-09-25 21:15 | Unit C | Unit B | 60 s | 53.91–54.05 V |
| 2026-09-26 21:15 | Unit B | Unit C | 60 s | 53.95–54.08 V |
| 2026-09-27 21:15 | Unit C | Unit A | 60 s | 53.93–54.06 V |
| 2026-09-28 21:15 | Unit A | Unit C | 60 s | 53.91–54.08 V |
| 2026-09-29 21:15 | Unit C | Unit B | 60 s | 53.93–54.06 V |
| 2026-09-30 21:15 | Unit B | Unit C | 60 s | 53.95–54.10 V |

Unit C appears every other day because it began the observation with materially
less accumulated runtime. Units A and B alternate in the other position. This
is the expected result of selecting the least-used parked unit while requiring
the currently carrying unit to hand off.

## Runtime persistence boundary

The recorder contains two genuine controller restarts on 2026-09-20: the
per-session runtime counters reset and the shelf re-enumerated. The current
serial-bound Home Assistant runtime ledger was enabled on 2026-09-21, after
those restarts. Later brief `unavailable` intervals did not reset the session
counters and therefore represent reconnects, not controller reboots.

The live controller's per-slot runtime inputs currently match the Home Assistant
records selected by serial, including after earlier slot reassignment. That
verifies steady-state serial mapping and periodic push behavior. It does **not**
yet prove restoration across a genuine controller reboot with the current ledger
enabled. A controlled restart would be required to close that final gap.

## Method

The analysis used Home Assistant's history API for the three parked sensors,
active-rectifier count, per-slot serial and runtime entities, persistent runtime
helpers, and rectifier output voltage. Installation-specific entity IDs and
hardware serial numbers are intentionally omitted.
