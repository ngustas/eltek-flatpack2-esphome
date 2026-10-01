# Eltek Flatpack2 CAN protocol and shelf operations

Bench-verified notes, established 2026-08-28/29 against two and extended
2026-09-20 with a third
**FLATPACK2 48/2000 HE** rectifiers (P/N 241115.105, rev 3.2 / 3.23, SW 2.00/2.00)
and a **Smartpack WEB/SNMP SW 4.7 / 03.07** controller.

Everything below was observed on the wire unless explicitly marked *inferred*.

The widely-circulated community protocol description was the starting point for
all of this and is accurate in the great majority of what it covers — login,
addressing, the status frame layout and its scaling all match exactly. A few
details differ from what these particular units do; those are noted as
observations rather than corrections, since firmware revision, model variant or
system configuration could each account for the difference. Where this document
and that one disagree, the sensible move is to check against your own hardware.

---

## 1. Bus

- 125 kbit/s, **29-bit extended identifiers**
- The bus is referenced to the **PSU negative output rail**. Connecting CAN
  without that common ground reference risks the transceiver.
- Rectifiers **current-share autonomously** by listening to each other's status
  broadcasts. No separate inter-rectifier protocol was observed: a capture with
  two units sharing contained only status, hello and controller traffic, with
  every frame accounted for. Sharing works with no controller present.

## 2. Identifier layout

    0x[CC][UU][FFFF]
      CC   device class   0x05 = rectifiers (a Smartpack also drives 0x01-0x0E)
      UU   unit id        0x01-0x3F, 0x00 = unaddressed, 0xFF = broadcast
      FFFF function

Both fields matter when searching for a command. Per-unit addressing works in
the `0x98xx / 0x9Cxx / 0x9FFC / 0xACxx / 0xBCxx / 0xBFFC` families; the
`0x4004` setpoint is the notable exception and is broadcast-only. An early probe
here varied only `UU` while holding `FFFF = 0x4004`, and concluded from that
alone that per-unit control did not exist — worth avoiding.

## 3. Message grammar

    payload = [op][selector][00][data...]
    response op = request op + 3   (+5/+6 for multi-byte variants)
    even op = read, odd op = write

Read ops seen: `0x08 0x18 0x20 0x28 0x50 0x60 0xC8`.
Multi-frame string replies carry a descending sequence counter, bit 7 set on the
first chunk (e.g. `86, 05, 04, 03, 02, 01`).

**Encodings vary between parameters — check each one:**

| Quantity | Encoding |
|---|---|
| Output voltage (`0x9FFC` sel `01`) | centivolts, **big-endian** |
| Output current (`0x9FFC` op `0x18` sel `00`) | deci-amps, single byte |
| Available max current (`0x9C00` sel `01`) | deci-amps, **little-endian** |
| Park voltage (`0xACxx`) | deci-volts, **big-endian** |
| Setpoint frame current | deci-amps, **little-endian** |
| Setpoint frame voltages | centivolts, **little-endian** |

## 4. Commands (controller → rectifier)

### Login — per unit
    0x05004800 | (id * 4)      payload = 6-byte serial + 00 00
The Smartpack sends serial + `14 0B` in the last two bytes; `00 00` also works.

### Set output — broadcast
    0x05FF4004   [I*10 LE][V_measured*100 LE][V_target*100 LE][OVP*100 LE]
    OVP 0x170B = 58.99 V (Smartpack) / 0x173E = 59.50 V

- **The current field is per rectifier**: the configured total divided by the
  number of active units. Observed live — 25 A configured with two units gave
  `7D 00` = 12.5 A; 14 A gave 7.0 A. The undivided total appears briefly (~1 s)
  before settling, which can be misleading in a short capture.
- **`FF FF` means no limit**, and is what the controller sends when current
  limitation is disabled.
- Field 2 carries *measured bus voltage* — the controller updates it as the bus
  ramps during walk-in. Field 3 is the target. Sending the target in both works.
- Per-unit forms `0x05<id>4004` produced no response on these units; that
  identifier is what the PSU transmits status on.

### Park / release — per unit  ← the efficiency-manager mechanism
    command   0x05XXAC02   7B 04 00 <V*10 BE> 01 01   park at that voltage
    response  0x05XXAC00   33 05 00 <V*10 BE> 01 01   from the RECTIFIER, 3-33 ms later
    release   0x05XXAC02   7B 04 00 00 00 01 01       same command, zero volts
                                                      (Smartpack sends it at unit 0)

- `01 C8` = 45.6 V parks a unit: it drops to 0.0 A, reports `Disabled` in
  PowerSuite, and its front LEDs go dark. `00 00` releases it.
- **Repeated every ~4 s as a keepalive rather than latched.** Stop sending and
  the unit resumes on its own after ~5 s — a sensible fail-safe, since a dead
  controller cannot leave rectifiers parked.
- **`AC00` is a response, not a command.** Isolated on the bench 2026-08-30 by
  sending each frame alone for 70 s:

      AC02 only   unit stays parked indefinitely; every request answered on
                  AC00 3-33 ms later (n=35, median 15 ms)
      AC00 only   nine consecutive frames, ZERO replies, and the rectifier
                  wakes and resumes carrying within ~10 s

  So a controller only ever needs to send `AC02`. Sending `AC00` is inert.

- **The release is acknowledged too.** Confirmed over two full rotations
  2026-08-30 (`captures/can-capture-rotation-final.log`): the
  zero-voltage `AC02` draws a zero-voltage `AC00` back, `33 05 00 00 00 01 01`,
  6-8 ms later. Across that capture the controller sent 25 `AC02` frames (17 to
  unit 1, 8 to unit 2) and received exactly 25 `AC00` frames, 17 and 8 — a
  clean 1:1 on a second, independent data set.

- **This corrects an earlier reading of the Smartpack capture.** What looked
  there like a per-unit command pair — `AC02` then `AC00`, 507 times with no
  exceptions — was one request and its reply. The invariable ordering was the
  giveaway in hindsight: responses always follow requests.

  Release reads the same way. The Smartpack sends a single zero-voltage `AC02`
  to unit zero (the broadcast address) and each rectifier answers on its own
  `AC00`. That is why those replies arrive in no fixed order — they are
  independent responses racing, not an enumerated command sequence.

- Because the Smartpack's release is broadcast, it drops *every* unit out of
  standby and re-parks any that should stay parked on the next ~4 s keepalive.
  **This controller deliberately diverges**: it addresses the release to one
  unit, so rotating one rectifier cannot disturb another that is still parked.
  Per-unit `AC02` is proven — it is what holds the park in the first place.
  Two back-to-back rotations under this scheme (2026-08-30) handed the load over
  without a gap: release of the incoming unit at t+0, park of the outgoing unit
  60 s later, and never more or fewer than one unit parked at any instant.

- A parked rectifier reports status `0x10` continuously, the same code as
  walk-in, while showing 0.0 A and the prevailing bus voltage. `0x10` is
  better read as "not regulating" than specifically "ramping".
- The park level appears to be the configured **Rectifier_Standby_Voltage**, a
  documented System Voltages parameter, 45.6 V on this shelf. It is meant to sit
  below the battery's end-of-discharge voltage.

### Write EEPROM default voltage
    0x05009C00   29 15 00 <V*100 LE>
Takes effect when the unit next logs out. Verified by writing 48.0 V and
power-cycling: the unit walked in to 48.03 V and held.

### Read available max current — per unit
    0x05XX9C00   request 20 01 00  →  response 23 01 00 <LE16 deci-amps>
    (0x05XX9FFC answers selector 0x01 the same way)

This value is **real-time and input-limited**, not a fixed derate. Measured on
this shelf, both units agreeing to within 0.3 A:

| Line voltage | Advertised |
|---|---|
| 117-118 VAC | 25.4 - 25.7 A |
| 122 VAC | 26.7 - 26.9 A |
| 244-246 VAC | **42.2 A** |

The nameplate reads 53.5 V / 37.4 A with **AC current 12.5 A max**, and the low
line figures follow from it: 12.5 × 120 × ~0.95 ÷ 53.5 ≈ 26.6 A. The 240 V
result does not. It overshoots the 37.4 A nameplate and lands on **42.2 A**,
which is the 42.3 A that `0x05XX9FFC` op `0x20` sel `0x15` reports — settling
what that second number is. It is the unit's absolute ceiling, and on high line
it becomes the binding constraint: the AC limit would allow 12.5 × 244 × 0.95 ÷
55.1 ≈ 52.6 A, well past it, so the hardware ceiling wins.

Practical consequence: the same rectifier delivers **66% more current on 240 V**
than on 120 V. Any shelf sizing must poll this value rather than trust the
label — on 120 V a nameplate-based estimate is 64% high, which sizes the shelf
for a load it cannot carry.

### Read identity — per unit  ← model, part number, build date
    0x05XXBC00   request 50 <sel> 00  →  response 53 <sel> <seq> <data...>

Op `0x50` is the identity channel, and it is the only place the units name
themselves. String replies are chunked like the serial: bit 7 set on the first
frame, the counter then descending to 1.

| sel | Meaning | Encoding |
|---|---|---|
| `0x00` | Model | ASCII, `FLATPACK2 48/2000 HE` |
| `0x04` | Eltek part number | ASCII, e.g. `241115.105` |
| `0x14` | Build year | uint16 **little-endian** (`E3 07` = 2019) |
| `0x18` | Build month | single byte |
| `0x1C` | Build day | single byte |

Measured across three units on this shelf, 2026-09-20:

| Unit | Model | Part no. | Build date |
|---|---|---|---|
| A | FLATPACK2 48/2000 HE | *not reported* | 2013-04-20 |
| B | FLATPACK2 48/2000 HE | *not reported* | 2014-03-19 |
| C | FLATPACK2 48/2000 HE | `241115.105` | 2019-09-26 |

**No software-version field was established.** Nothing decoded on `0xBC00`,
`0xBFFC`, `0x9C00` or `0x9FFC` returned a verified software version under the
read operations tried (`0x08 0x20 0x50 0xC8`). What distinguishes generations
in these captures is which selectors answer at all:

- The 2019 unit answers `0x04`, `0x08` and `0x0C`; the 2013 and 2014 units
  answer none of the three and return the model only. Selector `0x04` is the
  verified part number. The meanings and encodings of `0x08` and `0x0C` remain
  unresolved; do not label `0x0C` as firmware based only on a value resembling
  a hardware revision.
- The 2019 unit does **not** acknowledge the park command on `0x05XXAC00`,
  while both older units reply every time. It parks correctly regardless -
  0.0 A and status `0x10` - so the ack is informational, not a handshake. Do
  not treat a missing `AC00` reply as a failed park.

The serial's leading four digits are a year-and-week stamp that roughly tracks
the build date: `1316…` = 2013 w16 against a 2013-04-20 build (w16, exact),
`1938…` = 2019 w38 against 2019-09-26 (w39). `1409…` = 2014 w09 against a
2014-03-19 build (w12) is the loosest of the three, so treat it as approximate.

### Read warnings / alarms
    0x05XXBFFC   08 04 00 (warnings) / 08 08 00 (alarms)
    response starts 0E, 16-bit field in bytes 3-4

### Enumeration prompt — asks every unit to announce itself
    0x0500BC02   6D 30 00
Every rectifier replies within 0–12 ms with `0x05XX4400` carrying its serial,
**even while logged in**. Observed identically across three controller boots.
This is a clean way to identify units that would otherwise stay silent (§6).

## 5. Telemetry (rectifier → controller)

    0x05XX40YY   [inlet degC][I*10 LE][V*100 LE][Vin volts LE][outlet degC]
      YY = 04 normal | 08 warning | 0C alarm | 10 walk-in

### Interpreting state `0x08`

The `08` state means that at least one warning indication is active; it does not,
by itself, describe the severity of the condition. On these units, warning bit 7
(`0x0080`) is Current Limit and is asserted when a rectifier reaches its
commanded limit. That may be entirely expected — for example, when the
controller deliberately limits battery-charging current. In that situation it is
useful operating information rather than a fault. The same indication may
deserve attention if it is unexpected, caused by derating, or preventing the
shelf from supporting its load. Decode the underlying warning bits and interpret
them in the context of the commanded limit and actual system demand.

### Other telemetry notes

- At near-zero output the voltage bytes decode to implausible values (~652 V
  seen during a bus collapse). Worth clamping, or it inflates any power figure
  derived from it.
- A parked unit keeps reporting, and its state byte alternates between `04` and
  `10` while sitting at 0.0 A. Output current, rather than the state byte,
  indicates whether a unit is parked.

### Warning / alarm bit field

These names come from Eltek's PowerSuite "Detailed rectifier status" table and
match the WebPower firmware, which contains the same table as HTML cells
`al0`–`al15` in bit order:

    0 OVS Lockout        4 Lo Mains       8 Internal Voltage  12 Low Output Voltage
    1 Mod Fail Primary   5 Hi Temp        9 Module Fail       13 Sub Module Fail
    2 Mod Fail Secondary 6 Low Temp      10 Fan1 speed Low    14 Fan3 speed Low
    3 Hi Mains           7 Current Limit 11 Fan2 speed Low    15 (blank/unused)

Bits 10–12 differ from the community description, which lists Mod Fail Secondary
a second time and does not include Low Output Voltage. Bit 7 = Current Limit is
confirmed empirically here. Bit 15 is labelled "Inner volt" by PowerSuite but
left blank and marked unused by the WebPower firmware.

## 6. Discovery and login lifecycle

    0x0500XXXX   hello, payload 1B|1C + 6-byte serial   (XXXX derives from serial)
    0x05XX4400   login request, payload = bare serial   ~16 s while logged out

- Both `0x1B` and `0x1C` prefixes appear. Units under Smartpack control were seen
  emitting `0x1C`, so a filter matching only `0x1B` will miss them.
- Login expires after roughly 15 s, and **any addressed traffic refreshes it —
  including the broadcast setpoint**, not only login frames. An earlier test here
  concluded login never expires; it had gated only the login path while the
  allocator kept broadcasting, so the units were never actually starved.
- The practical consequence: a controller that begins broadcasting immediately
  after a restart keeps previously logged-in units from ever announcing
  themselves, so their serials remain unknown. Sending the enumeration prompt
  (§4) resolves this without waiting for a timeout.
- The rectifier **stores its own assigned ID alongside its serial**, so a
  reinserted unit is recognised and given the same ID. IDs assigned by any
  controller persist in the hardware.
- Walk-in from ~44 V to float takes ~5–6 s (Short setting) or 60 s (Long). Total
  recovery from a park release is **~11 s** — roughly 5.4 s for the keepalive to
  lapse plus a ~6 s ramp.

## 7. Efficiency management (Smartpack behaviour)

Documented rules, all observed in practice:

- Sheds when total load is below **~50% of installed capacity**; the target band
  is **50–80%** of each running rectifier's output.
- **Redundant mode keeps ideal + 1 running**, and with load under 2 kW it keeps
  two units on — so a two-unit shelf will not shed in Redundant mode.
- **Shuffle Time** (hours) rotates duty. **"Rectifier Off" delay** (minutes) is
  the make-before-break overlap: the incoming unit is released first and the
  outgoing one parked after the interval.
- **Test mode rescales the timers** — shuffle hours→minutes, off-delay
  minutes→seconds. It is distinct from the battery Test Mode of the same name in
  the glossary.
- **Runtime is not published on the bus.** No per-unit runtime parameter was
  found (only the generator has Run Hours). The controller tracks it internally,
  which it can do because it issues the park keepalives.
- **Four events halt the manager** and turn all units on: a rectifier error or
  comms loss, a change in the installed rectifier count, a change to system
  voltages, or a battery Boost/Test starting. Recovery is a rectifier recount.
- **An active Rectifier Current Limitation inhibits the manager entirely**, which
  the Summary tab reports as `Efficiency manager status: Current limit`. This is
  an easy one to miss when a correctly-configured manager appears to do nothing.

## 8. Operational notes

- Smartpack front keypad **Service Options password `0003`** (factory default).
  Three wrong attempts lock it out for a period, and the display returns to
  Status Mode after 30 s idle. The **rectifier recount** lives there (sec 17.3),
  and appears as "Reset Number of Modules" in PowerSuite/WebPower.
- **The Smartpack is powered from the DC bus it regulates.** On a bench with no
  battery, a break-before-make gap browns out the controller and resets its
  configuration — this quietly ended several Test mode sessions before the cause
  was clear.
- Such a gap also raises genuine rectifier alarms (`0x0C`), which then halt the
  efficiency manager. With a battery present the gap is invisible and neither
  happens.
- **Front-panel LEDs:** green on = powered, flashing = controller reading it,
  off = no mains. Red = shutdown (low mains, ≥75 °C, high output, CAN failure).
  Yellow = derating, current limit, or loss of comms with the controller.
  **All dark = mains failure or an efficiency-management park.**
- WebPower is a **protocol translator** — pComm over RS-232 at 38400 to the
  Smartpack, HTTP/SNMP to the network. Its firmware contains no CAN code; the
  Smartpack (03.07) owns the bus. pComm remains an unexplored path to the
  controller if something is ever needed that CAN cannot express.
- Home Assistant displays temperatures in Fahrenheit while the firmware works in
  Celsius — an inlet reading of 93.2 is 34 °C.

## 9. What our M5 controller implements

- Discovery via hello (`1B`/`1C`), login request, and the enumeration prompt at
  boot and every 30 s, with adoption by address as a fallback.
- Login refresh every 5 s for units still announcing.
- Broadcast setpoint on `0x05FF4004`, shelf current limit divided across active
  (non-parked) units — the same semantics as the Smartpack.
- Efficiency manager: 50/80% band, 30 s load averaging, 30 s shed hold, optional
  redundancy, make-before-break rotation with a configurable overlap.
- Park and release via `0x05XXAC02` at 45.6 V on a 3.5 s keepalive; `AC00`
  is the rectifier's response, not a controller command.
- Per-unit capability polled every 30 s so capacity tracks input voltage.
- Named fault decode, voltage sanity clamp, and parked units continue reporting.

**Verified 2026-08-29:** shed after 30 s; rotation with the bus holding
54.26–54.33 V throughout — no sag, no alarms, and duty genuinely swapped.

**Live automatic rotation, 2026-09-21 through 2026-09-30 local time:** Home
Assistant recorder history contains ten consecutive 24-hour three-unit
handovers. Each transition ran the incoming unit alongside the outgoing unit
for exactly 60 seconds (`1 → 2 → 1` active rectifiers), never reached zero
active units, and held reported output voltage within 53.91–54.10 V in the
four-minute observation window around each handover. All three units
participated in the runtime-ordered sequence. See
[`evidence/automatic-rotation-live-history.md`](../evidence/automatic-rotation-live-history.md).

**Live three-rectifier cycle, 2026-09-20 local time:** three distinct serials
were communicating on 233–238 VAC, each advertising 42.2 A available. With
the XW forcing grid AC disqualified and the shelf limit at 15 A, the M5 started
with one running and two parked. Switching efficiency off released both parked
units; they entered walk-in, then all three supplied current (roughly 2–4 A
each at an 8–9 A shelf load). Switching it back on parked two after roughly
33 s, leaving one carrying. Enabling redundancy woke one parked unit and
settled at two running / one parked, about 7.5 A each at a 15 A shelf load;
turning redundancy back off shed to one running / two parked after roughly
33 s. The test restored efficiency on and redundancy off.

These times are from Home Assistant polling every ~2 s, not a direct CAN
timestamp. A release changes the controller's running count before the waking
rectifier supplies output; after the redundancy wake it took about 14 s before
the newcomer carried substantial current. During that walk-in the measured
shelf output briefly reached 23.7 A despite the 15 A requested limit, consistent
with the broadcast setpoint overlap described above. Rectifier 1 voltage in the
capture ranged 53.62–54.06 V as the XW load changed; only Current Limit and
None appeared in fault text. The individual `Online` sensors remained on for
parked units because they still communicate. See
[`evidence/three-rectifier-efficiency-anonymized.csv`](../evidence/three-rectifier-efficiency-anonymized.csv)
for anonymized states, voltage, and per-unit output current. This confirms the
three-unit parking and redundancy states; it does not constitute a controlled
efficiency or high-load capacity measurement.

### Known gaps

- Per-serial runtime helpers now exist in HA; this three-unit cycle did not test
  automatic duty rotation or persistence after a controller reboot.
- The controller never sends `FF FF`, so "no limit" cannot be expressed.
- Bench instruments remain in the firmware (CAN sniffer, parameter sweep,
  per-unit method probe, observe mode) — remove or guard before production use.
- The `0x1C` hello path is implemented but not yet exercised; it needs a
  Smartpack on the bus to produce one.
