# Derived evidence

Raw logs are intentionally not included: several contain local IP addresses,
hardware serial numbers, unrelated controller traffic, and megabytes of repeated
status frames. This directory keeps the reproducible conclusions without those
installation details.

## Three-rectifier efficiency cycle

`three-rectifier-efficiency-anonymized.csv` is the 2026-09-20 Home Assistant
capture with the three rectifier serials replaced by `unit-a` through `unit-c`
and unrelated inverter mode/load fields removed.

The approximately two-second samples show:

- initial state: 1 running / 2 parked at an 8.2–8.3 A shelf load;
- efficiency disabled: all three released and carrying after walk-in;
- efficiency enabled: return to 1 running / 2 parked after about 33 seconds;
- redundancy enabled: 2 running / 1 parked;
- redundancy disabled: return to 1 running / 2 parked after about 33 seconds;
- every unit advertised 42.2 A available on 233–238 VAC;
- a transient 23.7 A shelf reading during the redundancy wake despite a 15 A
  requested limit, demonstrating the broadcast-setpoint overlap;
- parked units remained online and continued reporting bus voltage.

This confirms state transitions and three-unit parking behavior. It is not a
controlled efficiency study, high-load capacity test, or direct CAN-timestamped
measurement.

## Frame-level capture conclusions

The retained raw captures in the private work area were reduced to these counts:

- `AC02` isolation test: repeated addressed `AC02` held a unit parked; `AC00`
  alone produced no replies and the unit resumed carrying.
- final two-unit rotation: 25 transmitted `AC02` frames and 25 matching `AC00`
  replies on the two older units (17 for unit 1, 8 for unit 2), including release
  acknowledgements 6–8 ms after the request.
- the later 2019 unit parked correctly but produced no `AC00` acknowledgement.
- enumeration prompt `0x0500BC02 / 6D 30 00` caused all three installed units to
  announce on their addressed `0x05XX4400` frames within 0–12 ms.
- two-unit make-before-break rotation held the bus between 54.26 and 54.33 V,
  with no sag or alarm.
