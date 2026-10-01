# Installation and operation

## Hardware

The supplied configuration targets:

- M5Stack Atom Lite (ESP32)
- M5Stack ATOMIC CANBus Base using the isolated CA-IS3050G interface
- up to four Eltek Flatpack2 rectifiers
- ESPHome 2026.8.0 or newer

The configuration uses GPIO 22 for CAN TX, GPIO 19 for CAN RX, and GPIO 27 for
the Atom status LED. Change the substitutions only if your board is wired
differently.

## CAN wiring

The observed bus is 125 kbit/s and uses 29-bit extended identifiers. Connect
CAN-H and CAN-L with correct termination for the physical topology. The bus is
referenced to the rectifier negative output rail; follow the isolated interface
manufacturer's reference/ground instructions. Do not assume CAN-H and CAN-L are
the only required connections.

Disconnect mains and isolate stored DC energy before changing wiring. Fuse the
DC output and size conductors for the available fault current, not merely the
intended charge current.

## Configure ESPHome

Copy the example secrets file:

```text
esphome/secrets.example.yaml -> esphome/secrets.yaml
```

Set Wi-Fi, API encryption, and OTA credentials. `secrets.yaml` is ignored by
Git. Then edit the substitutions at the top of
`esphome/m5stack-eltek-shelf.yaml`:

- `float_voltage`: normal shelf target.
- `ovp_voltage`: over-voltage value sent in each broadcast.
- `fallback_voltage`: EEPROM default used when controller traffic stops. An
  explicit button press writes it; changing YAML alone does not modify a unit.
- `voltage_min` / `voltage_max`: Home Assistant control bounds.
- `shelf_current`: total requested steady-state shelf current.
- pin substitutions if using different hardware.

The checked-in numbers describe one 16S LiFePO4 installation at 3.40 V/cell.
They are examples only. Derive all limits from the cell chemistry, series count,
BMS limits, wiring, fusing, and load behavior.

Validate and compile:

```powershell
esphome config esphome/m5stack-eltek-shelf.yaml
esphome compile esphome/m5stack-eltek-shelf.yaml
```

For a first serial flash, use ESPHome's normal `run` or `upload` workflow. Do
not perform a first power-up against a battery until configuration validation,
CAN polarity, termination, and voltage limits have been checked independently.

## Commissioning sequence

1. Power the controller with CAN disconnected and confirm it boots.
2. Enable **Observe Mode** before connecting it to a shelf controlled by a
   Smartpack or another controller. Observe Mode suppresses controller traffic.
3. Connect CAN and verify status frames, input voltage, output voltage, and unit
   count. Temperatures are reported in Celsius by firmware; Home Assistant may
   display a converted unit.
4. Remove the old controller or otherwise ensure there is only one active
   controller before disabling Observe Mode.
5. Start with efficiency management off. Confirm every installed unit is
   discovered, online, and carrying.
6. Set a conservative shelf current. Confirm `Allocated Current Per Rectifier`
   equals the total divided by the carrying count, limited by the weakest unit's
   advertised capability.
7. Only after stable operation, enable efficiency management and test a park and
   release at low load. Verify actual output current, not merely `Online`.

Efficiency management defaults off after a fresh flash. Redundancy defaults on
but has no effect until the manager is enabled.

## What the controller does

### Discovery and login

The controller accepts both `0x1B` and `0x1C` hello forms, processes addressed
login requests, and sends `0x0500BC02 / 6D 30 00` after boot and every 30 seconds.
That prompt makes already logged-in units announce their serials. Known logins
are refreshed every five seconds; a missing slot is not reused until the old
login should have expired.

### Current allocation

`Shelf Current Limit` is a total. The controller divides it among rectifiers
that are online, unparked, and past walk-in. It polls each unit's real-time
available maximum every 30 seconds and clamps the common share to the weakest
carrying unit. The resulting `Effective Shelf Current Limit` can therefore be
lower than requested.

The output command is broadcast-only. During a planned wake/rotation, incumbents
keep their prior share until the incoming rectifier has settled. This protects
bus continuity but temporarily increases the aggregate ceiling. Use an upstream
BMS/charger interlock if a hard no-overshoot current boundary is mandatory.

### Efficiency, redundancy, and rotation

The manager uses a 30-second load average and a 50–80% target band. It wakes
capacity immediately and requires a sustained low-load condition before shedding.
Redundancy adds one running unit above the calculated ideal when available.

Parking sends addressed `AC02` commands every 3.5 seconds. If the ESP32 stops,
keepalives expire and parked modules return to service. Duty rotation releases
the incoming module first and waits for it to carry before parking the outgoing
module. The overlap is configurable and should exceed the observed recovery
time; about 11–14 seconds was seen, while the test configuration used 60 seconds.

Runtime ranking is serial-bound in the optional Home Assistant helpers. The
rectifiers do not publish lifetime runtime.

### State and faults

Key status interpretations are:

- `0x04`: normal regulating state.
- `0x08`: warning active. Bit 7 (`0x0080`) is Current Limit and can be expected.
- `0x0C`: alarm.
- `0x10`: not regulating / walk-in; also seen while parked.

Parked status is determined from controller intent plus output current, not the
state byte alone. The firmware range-checks implausible voltage telemetry before
using it in power calculations.

## Home Assistant

ESPHome exposes shelf controls and aggregates plus per-slot current, voltage,
input voltage, temperatures, serial, capability, state, fault text, online, and
parked status. Important controls include:

- Voltage Setpoint and Shelf Current Limit
- Efficiency Manager Enabled and Efficiency Redundancy
- Shuffle Time Hours, Transition Time Minutes, Park Voltage, and Shuffle Now
- Observe Mode, CAN Sniffer Enabled, and TX Log Enabled
- Default Voltage plus the explicit EEPROM-write button

Home Assistant generates entity IDs from the device and entity names. The sample
package and dashboard use the default `m5stack_eltek_shelf` prefix; edit them if
your registry retained older or customized IDs.

Copy `examples/home-assistant/eltek-shelf-package.yaml` into your packages
directory if you want typed controls and a four-record scaffold for serial-bound
runtime totals. The released package does not accumulate or push those totals:
that mapping is installation-specific and must only increment a record while its
matching serial is online and not parked. Add more record pairs if rectifiers may
move through the shelf over time. Copy the six control-automation blocks into
`automations.yaml`, then paste the Lovelace example as a manual card. These
examples are optional; the controller continues maintaining the shelf if Home
Assistant is unavailable.

## Troubleshooting

- **No frames:** verify 125 kbit/s, extended identifiers, CAN-H/CAN-L polarity,
  termination, transceiver power, and the DC-negative reference.
- **Status but no serial:** confirm the enumeration prompt is transmitted and
  accepted. Broadcast setpoints can keep old logins alive indefinitely.
- **Current limit is too low:** inspect each unit's advertised maximum. The same
  tested module produced roughly 26 A at 120 VAC and 42.2 A at high line.
- **Unit stays online while parked:** expected. It still reports telemetry.
- **No `AC00` reply:** the 2019 test unit never acknowledged but parked normally.
  Check current and state rather than treating the reply as a handshake.
- **Manager will not shed:** current limitation, faults, communication loss,
  changing unit count, or battery boost/test can inhibit Smartpack-style
  efficiency operation. Also check redundancy and the 30-second hold.
- **Rotation never completes:** increase the overlap and verify the incoming unit
  exits walk-in and carries current before the readiness timeout.
- **Unexpected high aggregate current during wake:** this is the documented
  broadcast-setpoint overlap. Disable efficiency management if unacceptable.
- **Voltage jumps after controller loss:** the rectifier reverted to its EEPROM
  default. Verify and deliberately program a battery-safe fallback.

## Production checklist

- Keep Observe Mode and bench probe controls inaccessible to casual operators.
- Leave sniff/TX logging disabled except during diagnosis.
- Back up Home Assistant runtime helpers if rotation order matters.
- Test loss of Wi-Fi and Home Assistant; neither should reboot the power loop.
- Test loss of ESP32 power; all parked units should recover automatically.
- Test a pulled rectifier, changed unit count, and alarm response at safe load.
- Revalidate every voltage whenever battery chemistry or series count changes.
