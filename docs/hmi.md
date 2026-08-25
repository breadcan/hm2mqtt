# HMI Inverter MQTT Document

Covers the Marstek HMI micro-inverter family:

- **MI800** — sold as the Marstek Saturn (model code `MST-MI0800`), 2 PV
  inputs, 800W output. Reports as device type `HMI-1`. Marstek also lists a
  higher-power `MST-MI1000` sibling in the same product line; this document
  is based entirely on a real `MST-MI0800`; the MI1000's device type and
  whether it shares the same protocol have not been verified.
- **HMI-2000** — the 4-PV, 2000W-class variant of the same product line, 4 PV
  inputs. Reports as device type `HMI-2000`.

Both share the same protocol; where they differ, the section says so.

This document is community-reverse-engineered (see [GitHub issue
#115](https://github.com/tomquist/hm2mqtt/issues/115) for the original
research thread), not sourced from Marstek. Confidence in each field varies
and is called out explicitly below — treat anything not marked "confirmed" as
a best guess.

## Table of Contents

1. [Device types](#1-device-types)
2. [Subscribe to your device](#2-subscribe-to-your-device)
3. [Read device information](#3-read-device-information)
   1. [Public](#31-public)
   2. [Receive](#32-receive)
4. [Set maximum output power](#4-set-maximum-output-power)
   1. [Public](#41-public)
5. [Set mode](#5-set-mode)
   1. [Public](#51-public)
6. [Set grid connection ban](#6-set-grid-connection-ban)
   1. [Public](#61-public)
7. [Error fields](#7-error-fields)

## 1 Device types

| Device type | Family | Sold as |
| --- | --- | --- |
| `HMI-1` | MI800 | Marstek Saturn MI0800, 2 PV inputs |
| `HMI-2000` | HMI-2000 | Marstek Saturn 2000-class, 4 PV inputs |

## 2 Subscribe to your device

Before sending/receiving messages in MQTT, you must subscribe to your device using the following command:

```
hame_energy/{type}/device/{uid or mac}/ctrl
```

Commands are published to:

```
hame_energy/{type}/App/{uid or mac}/ctrl
```

Newer devices/firmware also use a `marstek_energy/` prefix in place of
`hame_energy/`, with a different device ID scheme — see the "Topic structure
— Marstek cloud broker" section of [docs/b2500.md](./b2500.md) for how that
ID is derived. hm2mqtt subscribes to both prefixes for this device family.

## 3 Read device information

### 3.1 Public

Topic:

```
hame_energy/{type}/App/{uid or mac}/ctrl
```

Payload:

```
cd=1
```

### 3.2 Receive

You will receive a message, such as:

```
ele_d=11,ele_s=1433,ele_m=1433,pv1_v=334,pv1_i=0,pv1_p=16,pv1_s=1,pv2_v=335,pv2_i=0,pv2_p=15,pv2_s=1,pe1_v=17,fb1_v=847,fb2_v=826,grd_f=4999,grd_v=2455,grd_s=1,grd_o=29,chp_t=33,rel_s=1,err_t=0,err_c=0,err_d=0,ver_s=120,mpt_m=1,ble_s=1,mpt1=1,mpt2=1,wif_r=69,fc4_v=202406141323,gc=0,pl=800,ct_r=0,ct_f=0,ct_c=0
```

(a real MI0800 capture, firmware 120 — note `ele_w` is already absent here, matching the note below)

The HMI-2000 (4-PV) variant additionally reports `pv3_*`/`pv4_*`/`fb3_v`/`fb4_v`
fields for the extra two inputs, plus `ele_h`, `ht` and `wf` (meaning
unconfirmed, not currently seen on 2-PV units).

Description of the above parameters:

| Field | Description |
| --- | --- |
| `ele_d` | Daily energy generated, in 0.01 kWh |
| `ele_w` | Weekly energy generated, in 0.01 kWh. Seen on early firmware only - not present at all in every sample checked from firmware 107 onward, cause unknown |
| `ele_m` | Monthly energy generated, in 0.01 kWh |
| `ele_s` | Total (lifetime) energy generated, in 0.01 kWh |
| `ele_h` | HMI-2000 only. Unknown, possibly hourly energy generated (unconfirmed) |
| `pv1_v`, `pv2_v`, `pv3_v`, `pv4_v` | PV input N voltage, in 0.1V. PV3/PV4 on HMI-2000 only |
| `pv1_i`, `pv2_i`, `pv3_i`, `pv4_i` | PV input N current, in 0.1A |
| `pv1_p`, `pv2_p`, `pv3_p`, `pv4_p` | PV input N power, in W |
| `pv1_s`, `pv2_s`, `pv3_s`, `pv4_s` | PV input N active status (`1` = active) |
| `pe1_v` | Unknown. Not exposed by hm2mqtt |
| `fb1_v`, `fb2_v`, `fb3_v`, `fb4_v` | Unknown, guessed to be an MPPT regulation-loop feedback voltage per PV channel - unconfirmed. Not exposed by hm2mqtt |
| `grd_f` | Grid frequency, in 0.01 Hz |
| `grd_v` | Grid voltage, in 0.1V |
| `grd_s` | Grid connected status (`1` = connected) |
| `grd_o` | Grid output power, in W |
| `chp_t` | Chip temperature, in °C |
| `rel_s` | Unknown, guessed to be the grid relay's open/closed status (`1` = closed) - unconfirmed. Not exposed by hm2mqtt |
| `err_t` | Error type. See [section 7](#7-error-fields) |
| `err_c` | Error count. See [section 7](#7-error-fields) |
| `err_d` | Error details. See [section 7](#7-error-fields) |
| `ver_s` | Firmware version |
| `mpt_m` | Currently set mode. See [section 5](#5-set-mode) |
| `mpt1`, `mpt2` | Unknown, guessed to be per-channel MPPT-related flags - unconfirmed. Not exposed by hm2mqtt |
| `ble_s` | Not a live Bluetooth signal-strength reading - see [section 7](#7-error-fields) for what's actually been observed |
| `wif_r` | Wi-Fi RSSI, in dBm. Field reused from the CT002/CT003 meters' protocol; not independently re-verified for this device family |
| `fc4_v` | FC41D communication-module firmware version |
| `gc` | Grid connection ban state (`1` = banned). See [section 6](#6-set-grid-connection-ban). Not present on every firmware version seen (e.g. absent on some firmware 107 units) |
| `pl` | Configured maximum output power, in W. See [section 4](#4-set-maximum-output-power). Same firmware-dependent absence as `gc` |
| `ct_r`, `ct_f`, `ct_c` | Unknown. Not exposed by hm2mqtt |
| `ht`, `wf` | HMI-2000 only. Unknown. Not exposed by hm2mqtt |

## 4 Set maximum output power

### 4.1 Public

Topic:

```
hame_energy/{type}/App/{uid or mac}/ctrl
```

Payload:

```
cd=8,p1={watts}
```

Confirmed via live testing against a real device (issue #115): the device
acknowledges with `cmd=8 ok`, and the app UI reflects the new value. The
configured value is read back as `pl` in the device information.

## 5 Set mode

### 5.1 Public

Topic:

```
hame_energy/{type}/App/{uid or mac}/ctrl
```

Payload:

```
cd=11,p1={mode}
```

| `p1` | Mode |
| --- | --- |
| `0` | Default / standard |
| `1` | "B2500 Boost" - this is the label the Marstek app itself uses for this mode, sniffed directly from real app traffic (issue #115), but what it actually changes on this device is unknown even to the device owner who found it |
| `2` | Reverse current protection |

Confirmed via live testing against a real device (issue #115): commands were
captured directly from the Marstek app's own MQTT traffic. The current mode
is read back as `mpt_m` in the device information.

## 6 Set grid connection ban

### 6.1 Public

Topic:

```
hame_energy/{type}/App/{uid or mac}/ctrl
```

Payload:

```
cd=22,p1={0 or 1}
```

`p1=1` bans grid connection (opens the grid relay); `p1=0` allows it.
Confirmed via live testing against a real device (issue #115), sniffed from
real app traffic the same way as [section 5](#5-set-mode). The current state
is read back as `gc` in the device information.

## 7 Error fields

`err_t`, `err_c` and `err_d` were never documented by Marstek and were added
to hm2mqtt in 2025-07 as a straight pattern-match from field names in a single
raw sample, with no decode table (issue #115). The notes below come from a
2026-08 investigation against a real MI0800 (2-PV, only PV1 physically wired
to a panel) and should be read as a work in progress, not a settled spec.

**`err_t` (error type)** is transmitted as a plain decimal integer, but
evidence points to it actually being the device's internal hex fault-code
register, serialized without converting it to decimal first. The official
Marstek MST-MI series manual documents a fault-code table (codes like `529`,
and hex-notated codes like `40A`-`41B`, implying the underlying register is
hex). Two codes have been cross-checked against real, independently-known
conditions on a live device and can be treated as confirmed:

| Raw `err_t` value | Hex | Manual meaning | Confirmed how |
| --- | --- | --- | --- |
| `1321` | `0x529` | PV-1 Input Undervoltage | PV1 (the only wired port) was at ~19-21V, below the ~22V startup threshold |
| `1297` | `0x511` | PV-2 No Input | PV2 (never wired to a panel at all) - exact match to a known, deliberate ground truth |

A third transition was observed to raw value `1062` (hex `0x426`), coinciding
with `err_c` incrementing and the grid relay opening (`rel_s` 1→0) - a
real fault event, but `0x426` doesn't appear in the manual's documented
range and remains unidentified. The rest of the manual's table has not been
independently verified against this device at all.

**`err_c` (error count)** stayed at `0` through an extended period where
`err_t` reported a PV-input condition that was really just low panel voltage
at dusk, and only incremented once a real fault happened (the grid-relay-open
event above). This suggests the device's own firmware already distinguishes
an informational condition from a counted fault, which is a more reliable
signal for "is this actually a problem" than trying to classify `err_t`
codes by severity. **Not yet confirmed whether `err_c` ever
resets/decrements** after a fault clears, or is a lifetime counter.

**`err_d` (error details)** - no working theory survives scrutiny yet.
While `err_t` was `1321`/`1297` (the PV-input codes above), `err_d`'s raw
value closely tracked `pv2_v`'s raw value - but PV2 had no panel wired to it
at all, while PV1 (the port the active code actually named) did. That rules
out "the voltage of whichever channel the current error refers to." Once
`err_t` changed to the `0x426` grid-fault code, `err_d` stopped tracking
`pv2_v` and jumped to unrelated, larger values instead. Left undecoded.

**`ble_s`** was originally guessed to be a small Bluetooth
connection-state enum (values like `1`-`5` are the only ones anyone has
posted). A live device was observed reporting `13945` - four orders of
magnitude off - and holding that exact value with zero drift for 2.5 hours
straight, then dropping to `1` at the exact same moment as the `err_c`/relay
fault event above, before partially climbing and resetting again shortly
after (coinciding with a `wif_r` drop to `0`, suggesting a second, smaller
comms hiccup). The frozen-for-hours behavior rules out a live elapsed-time
counter; the resets-on-disruption behavior suggests it tracks something like
comms stability rather than a real-time signal reading, but what it counts
between resets is unresolved.
