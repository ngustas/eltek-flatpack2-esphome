# Eltek Flatpack2 multi-rectifier controller

An ESPHome controller and reverse-engineered CAN reference for operating up to
four Eltek Flatpack2 rectifiers in one shelf. The tested hardware is the
**Flatpack2 48/2000 HE** with an M5Stack Atom Lite and isolated ATOMIC CANBus
Base.

This release covers two distinct uses:

- **ESPHome / Home Assistant:** discovery, login, telemetry, shelf-wide voltage
  and current control, capability-aware current allocation, fault decoding,
  efficiency parking, redundancy, and make-before-break duty rotation.
- **Other controllers:** a wire-level description of the observed 125 kbit/s
  extended-ID protocol, including discovery, enumeration, broadcast setpoints,
  parking, acknowledgements, telemetry, and fail-safe timing.

> [!CAUTION]
> Rectifiers contain lethal mains voltage and can deliver destructive DC fault
> current. The voltage values in this repository are examples for one 48 V
> installation, not safe defaults for another battery. Use correct fusing,
> conductors, enclosure, isolation, pre-charge, and battery-specific limits.

## Start here

1. Read [Installation and operation](docs/INSTALLATION.md).
2. Edit every pack-specific substitution in
   [`esphome/m5stack-eltek-shelf.yaml`](esphome/m5stack-eltek-shelf.yaml).
3. Copy `esphome/secrets.example.yaml` to `esphome/secrets.yaml` and supply your
   own values.
4. Compile before connecting the CAN interface to a live shelf.
5. Commission in **Observe Mode** first, then enable transmission deliberately.

Developers implementing a controller outside ESPHome should begin with
[CAN protocol and shelf behavior](docs/PROTOCOL.md).

## Repository contents

| Path | Purpose |
|---|---|
| `esphome/m5stack-eltek-shelf.yaml` | Current four-slot ESPHome controller |
| `examples/home-assistant/` | Optional generic helpers, automations, and Lovelace card |
| `docs/INSTALLATION.md` | Hardware, wiring, deployment, operation, safety, and troubleshooting |
| `docs/PROTOCOL.md` | Observed CAN behavior, inference boundaries, and tested differences |
| `evidence/` | Anonymized three-rectifier telemetry and concise capture summaries |
| `CHANGELOG.md` | Release notes and known limitations |

## Verified scope

Bench work began 2026-08-28/29 with two units and was extended on 2026-09-20
with a third. The tested rectifiers all identified as Flatpack2 48/2000 HE; the
three examples were built in 2013, 2014, and 2019. Behavior may differ on other
models or firmware.

Verified results include:

- 125 kbit/s CAN with 29-bit identifiers and the DC negative rail as reference.
- Discovery by hello, login request, and an enumeration prompt that works while
  units are already logged in.
- One broadcast output setpoint whose current value is **per carrying unit**.
- Autonomous current sharing between rectifiers.
- Real-time capability polling: about 25.4–26.9 A at 117–122 VAC and 42.2 A at
  233–246 VAC on the tested units.
- Addressed park/release through `0x05XXAC02`; `AC00` is a rectifier response,
  not a second command.
- Park keepalive expiry that returns units to service if the controller stops.
- Two-unit bench rotation without bus sag or alarms, followed by ten consecutive
  automatic three-unit handovers in live Home Assistant history. Every live
  handover used a 60-second `1 → 2 → 1` make-before-break sequence, with no
  zero-active interval and observed output voltage of 53.91–54.10 V.
- Three-unit transitions among 1 running / 2 parked, 3 running, and redundant
  2 running / 1 parked states.

## Important limits

- Three-unit automatic rotation is verified. Serial-bound runtime restoration
  across an actual controller reboot remains unverified: the recorded controller
  reboots predate the current Home Assistant runtime ledger. High-load capacity
  also remains untested.
- During make-before-break walk-in, the broadcast-only setpoint can temporarily
  allow aggregate output above the requested shelf limit.
- A parked rectifier still communicates; `Online` is not the same as carrying.
- The 2019 unit parked correctly but did not return the `AC00` acknowledgement
  seen from both older units. A missing acknowledgement is not proof of failure.
- Runtime totals can be injected into the controller for rotation ordering;
  persistence and serial-to-record accounting belong in Home Assistant because
  the rectifiers do not report lifetime runtime. The included package is a
  helper scaffold, not a complete runtime-accounting automation.
- Diagnostic sniff/probe controls remain in the YAML for commissioning and
  reverse engineering. Keep them disabled in normal operation.
- The controller cannot express the Smartpack's `FF FF` “no current limit” mode.

## Attribution and license

The controller grew from the MIT-licensed work in
[taHC81/Eltek-Flatpack2-ESPhome](https://github.com/taHC81/Eltek-Flatpack2-ESPhome)
and protocol groundwork in
[the6p4c/Flatpack2](https://github.com/the6p4c/Flatpack2). New code and
documentation in this repository are released under the [MIT License](LICENSE).
Eltek and Flatpack are trademarks of their respective owner; this is an
independent community project.
