# Changelog

## v0.1.1 — 2026-09-30

Evidence-only patch release; controller behavior is unchanged.

### Verification update

- Confirms ten consecutive automatic three-unit handovers from live Home
  Assistant history, at the configured 24-hour cadence.
- Every handover raised the active count from one to two for exactly 60 seconds,
  then returned to one; no zero-active interval occurred.
- Observed output voltage across the ±2-minute handover windows remained between
  53.91 and 54.10 V.
- Confirms all three rectifiers participated and that the sequence followed the
  serial-bound runtime ordering.
- Narrows the remaining persistence limitation: the current Home Assistant
  runtime ledger has not yet crossed a genuine controller reboot. Earlier
  recorded reboots occurred before the ledger automations were enabled.

## v0.1.0 — 2026-09-30

Initial public release of the four-slot M5Stack/ESPHome shelf controller and the
bench-verified protocol reference.

### Included

- Serial-aware discovery, login lifecycle, and periodic enumeration.
- Broadcast voltage/current control with carrying-unit allocation.
- Per-unit capability polling and named warning/alarm decoding.
- Efficiency parking, fail-safe keepalive, redundancy, and make-before-break
  rotation.
- Optional Home Assistant control helpers, a four-record runtime-ledger
  scaffold, six setpoint-sync automations, and a generic Lovelace card.
- Anonymized three-rectifier transition data and derived capture evidence.

### Documentation reconciliations

- Corrects the older forum draft: `0x05XXAC02` is the park/release command and
  `0x05XXAC00` is the rectifier's echo response, not a second command.
- Removes an unsupported claim that selector `0x0C` is a firmware string. The
  tested 2019 unit answers additional identity selectors, but their meanings are
  not all established.
- Records that the broadcast current value is per carrying rectifier and that
  aggregate current can temporarily exceed the requested shelf total during
  make-before-break walk-in.
- Separates communication (`Online`) from load-bearing (`carrying`/`parked`).

### Known limitations

- Tested only on three Flatpack2 48/2000 HE units and one Smartpack generation.
- Serial-bound runtime restoration across a genuine controller reboot and
  three-unit high-load capacity remain unverified.
- Diagnostic probe/sweep controls remain present and must be treated as bench
  tools.
- “No current limit” (`FF FF`) is not exposed by the controller.
