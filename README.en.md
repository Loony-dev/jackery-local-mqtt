<p align="center">
  <a href="README.md">🇺🇦 Українська</a> · <b>🇬🇧 English</b>
</p>

<h1 align="center">Jackery SolarVault 3 local MQTT protocol</h1>

<p align="center">
  An unofficial description of how <b>Jackery SolarVault 3</b> stations talk over your own MQTT broker:<br>
  topics, requests, replies, every field with its unit, control commands, smart plugs and meters.
</p>

> [!IMPORTANT]
> **Data current as of 24 November 2026.**
> Checked against a live SolarVault 3 Pro Max AC and the [official Jackery Home Assistant integration](https://github.com/Jackery-Official/jackery) v2.0.0 (commit `3cd17ff`, 13 Nov 2026).
> We re-check roughly **once a quarter**, and after station firmware, Jackery app or official integration updates. Next planned check: **February 2027**.
> The check history is in [VERIFICATION.md](VERIFICATION.md).

> [!NOTE]
> This is an unofficial project, not affiliated with Jackery. Jackery may change the protocol in a firmware update without notice.

---

## Contents

- [Legend](#legend)
- [Connecting the station to a broker](#connecting-the-station-to-a-broker)
- [Topics](#topics)
- [Message format](#message-format)
- [Message types](#message-types)
- [Polling: how to get data](#polling-how-to-get-data)
- [Station state (type 2, reply to 25)](#station-state-type-2-reply-to-25)
- [System state (type 106, reply to 105)](#system-state-type-106-reply-to-105)
- [Updates and statistics (type 107, 23)](#updates-and-statistics-type-107-23)
- [Control commands (type 1)](#control-commands-type-1)
- [Smart plugs and meters (type 100–103)](#smart-plugs-and-meters-type-100103)
- [Errors (type 123)](#errors-type-123)
- [Value reference](#value-reference)
- [Useful formulas](#useful-formulas)
- [Finding the station on the network](#finding-the-station-on-the-network)
- [Quick check from a terminal](#quick-check-from-a-terminal)
- [Known unknowns](#known-unknowns)
- [Contributing](#contributing)
- [Sources](#sources)
- [License](#license)

## Legend

| Mark | Meaning |
| :-: | --- |
| ✅ | Verified on a live station: the message was seen and the value matches reality. |
| 📄 | Taken from the official Jackery integration. Not verified on a live station. |
| ❓ | The meaning is our guess based on observations. May be inaccurate. |

Sample messages are in [`examples/`](examples). Serial numbers, tokens, addresses and network names are replaced.

## Connecting the station to a broker

The station connects to your MQTT broker as a client. The Jackery cloud isn't needed for this.

1. Run any MQTT broker on your local network (Mosquitto, for example) with a username and password.
2. In the Jackery app **2.0.0+**, open the station settings → MQTT and enter the broker address, port, username, password and topic prefix (`hb` by default).
3. The same screen shows the **token** (8 digits). Every request to the station must include it.

✅ The station connects with a **client id equal to its serial number**, MQTT 3.1.1, keepalive 60 s.

✅ The station **publishes nothing on its own**: it only answers requests. To get data you have to ask for it regularly (see [Polling](#polling-how-to-get-data)).

## Topics

`<prefix>` is the prefix from the settings (`hb` by default), `<SN>` is the station serial number.

| Topic | Direction | Contents |
| --- | --- | --- |
| `<prefix>/device/<SN>/action` | you → station | ✅ requests and commands |
| `<prefix>/device/<SN>/status` | station → you | ✅ station state (type 2) |
| `<prefix>/device/<SN>/event` | station → you | ✅ system state (type 106), plugs and meters (type 101), errors (type 123) |

To listen to every station at once: `<prefix>/device/+/status` and `<prefix>/device/+/event`.

## Message format

All messages are JSON. A request:

```json
{
  "type": 25,
  "eventId": 0,
  "messageId": 1932,
  "ts": 1795556302,
  "token": "12345678",
  "body": null
}
```

| Field | Type | Description |
| --- | --- | --- |
| `type` | number | ✅ message type, see the [table](#message-types) |
| `eventId` | number | ✅ `0` for requests, `3` for control commands |
| `messageId` | number | ✅ any number. The station echoes it in the reply, so you can match replies to requests |
| `ts` | number | ✅ Unix time in seconds |
| `token` | string | ✅ the token from the Jackery app. Without the right token the station answers with error 401 |
| `body` | object or `null` | ✅ request parameters |

The station's reply has the same `type`, `eventId`, `messageId`, `ts`, plus:

| Field | Description |
| --- | --- |
| `deviceType` | ✅ device type. SolarVault 3 sends `3` (see the [reference](#devicetype)) |
| `softver` | 📄 firmware version. ✅ Our station doesn't send it in these messages |
| `body.cmd` | ✅ internal data set number: `106` in type 2, `120` in type 106, `110` in type 101 |

📄 The official integration also accepts "flat" messages without `body`, with fields at the top level. We haven't seen any.

## Message types

| `type` | Direction | Topic | What it is | Status |
| :-: | --- | --- | --- | :-: |
| `25` | request | action | Station state | ✅ |
| `2` | reply | status | Reply to `25`: battery, AC outlet, limits, Wi‑Fi | ✅ |
| `105` | request | action | Whole-system state | ✅ |
| `106` | reply | event | Reply to `105`: grid, work mode, energy flows | ✅ |
| `107` | from station | event | Partial update (`soc`, `workMode`) | 📄 |
| `23` | from station | event | Accumulated energy (kWh) | 📄 |
| `1` | command | action | Change settings, `eventId: 3` | ✅ |
| `100` | request | action | List of smart plugs or meters | ✅ |
| `101` | reply | event | Reply to `100` | ✅ |
| `102` | from station | event | Update for a single sub-device | 📄 |
| `103` | command | action | Turn a smart plug on/off | 📄 |
| `123` | from station | event | Error, e.g. a wrong token | ✅ |

> [!NOTE]
> The official integration also sends a `type 2` request ("read all settings"). ✅ Our station doesn't answer it. Type 2 data only arrives as the reply to `25`.

## Polling: how to get data

The station doesn't push data, so you have to request it.

| What | How often | Request |
| --- | --- | --- |
| Station state | every 5 s | `type 25`, `body: null` |
| System state | every 5 s | `type 105`, `body: null` |
| Plugs and meters | once a minute is enough | `type 100`, `body: {"devType": 6}` and `{"devType": 2}` |

📄 The official integration polls everything every 5 s and treats the station as offline after 60 s without any reply.

✅ Replies arrive in under a second.

## Station state (type 2, reply to 25)

Topic `status`, `body.cmd = 106`. Full example: [`examples/status-type2.json`](examples/status-type2.json).

### Battery

| Field | Unit | Description | Status |
| --- | --- | --- | :-: |
| `soc` | % | System charge. 📄 Per the integration, the average of the station and expansion batteries | ✅ |
| `batSoc` | % | Main unit charge | ✅ |
| `batInPw` | W | Battery charging power | ✅ |
| `batOutPw` | W | Battery discharging power | ✅ |
| `stackInPw` / `stackOutPw` | W | ❓ Power on the battery module side (a few watts off `batInPw`/`batOutPw`) | ✅ |
| `batNum` | count | Number of expansion batteries. `0` means the main unit only | ✅ |
| `batState` | — | ❓ Battery state. ✅ `1` while charging | ✅ |
| `cellTemp` | 0.1 °C | Battery temperature: `447` = 44.7 °C | ✅ |
| `maxCellTemp` / `minCellTemp` | 0.1 °C | Highest and lowest cell temperature | ✅ |
| `maxCellVolt` / `minCellVolt` | 0.1 mV | Highest and lowest cell voltage: `34160` = 3.416 V | ✅ |

> [!WARNING]
> The official integration shows `cellTemp` as degrees without dividing by 10. On our station the value is clearly in tenths: at `447` the battery was warm, not 447 °C.

### AC outlet (EPS, "off-grid")

| Field | Unit | Description | Status |
| --- | --- | --- | :-: |
| `swEps` | 0/1 | Outlet output enabled. Can be changed with a command | ✅ |
| `swEpsState` | 0/1 | 📄 Outlet status: `1` normal, `0` fault | ✅ |
| `swEpsOutPw` | W | Power drawn from the AC outlet | ✅ |
| `swEpsInPw` | W | Power fed in through the AC outlet | ✅ |

### Grid port (on-grid)

| Field | Unit | Description | Status |
| --- | --- | --- | :-: |
| `inOngridPw` | W | Power entering the station from the grid | ✅ |
| `outOngridPw` | W | Power the station sends out of the grid port | ✅ |

### Solar

Present on versions with MPPT. ✅ On the **AC** model (no solar inputs) all values are always `0`.

| Field | Unit | Description | Status |
| --- | --- | --- | :-: |
| `pvPw` | W | Total solar power | ✅ |
| `pv1`…`pv4` | object | `{"pvPw": W, "commState": 0/1}` for each input | ✅ |

### Limits and settings

| Field | Unit | Description | Status |
| --- | --- | --- | :-: |
| `socChgLimit` | % | Charge limit, 50–100. Can be changed with a command | ✅ |
| `socDischgLimit` | % | Discharge limit, 5–49. Can be changed with a command | ✅ |
| `socForceChg` | — | ❓ Forced charging | ✅ |
| `maxOutPw` | W | Grid-port output limit, 0–2500. Can be changed with a command | ✅ |
| `maxInvStdPw` | W | ❓ Inverter maximum: `2500` | ✅ |
| `maxGridStdPw` | W | ❓ Grid export maximum under the country standard: `800` | ✅ |
| `autoStandby` | 0/1/2 | Auto standby mode: 0 invalid, 1 standby, 2 on | ✅ |
| `pcsSetPw` | W | ❓ Inverter power setpoint | ✅ |
| `stackChgCurLimit` / `stackDischgCurLimit` | ❓ | ❓ Battery module charge/discharge current limits (`60` / `600`) | ✅ |
| `emsChgCurLimit` / `emsDischgCurLimit` | ❓ | ❓ The same from the energy management system | ✅ |
| `maxIotNum` | count | ❓ How many sub-devices can be attached: `8` | ✅ |
| `ability` | bitmask | ❓ Device capabilities: `3840` | ✅ |

### Network

| Field | Description | Status |
| --- | --- | :-: |
| `wname` | Wi‑Fi network name | ✅ |
| `wip` | Station IP address on Wi‑Fi | ✅ |
| `wsig` | Wi‑Fi signal, dBm | ✅ |
| `mac` | Wi‑Fi MAC address | ✅ |
| `ethPort` | ❓ Ethernet connected (0/1) | ✅ |
| `eip` | Ethernet IP address. `0.0.0.0` without a cable | ✅ |
| `emac` | Ethernet MAC address | ✅ |

## System state (type 106, reply to 105)

Topic `event`, `body.cmd = 120`. Full example: [`examples/event-type106.json`](examples/event-type106.json).

### State and mode

| Field | Description | Status |
| --- | --- | :-: |
| `stat` | Device state, see the [reference](#stat) | ✅ |
| `ongridStat` | **Is the grid up:** `1` grid present, `0` no grid | ✅ |
| `ctStat` | CT meter: `1` online, `0` offline | ✅ |
| `gridSate` | Link to the grid meter: `1` normal, `0` fault. The typo is in the firmware | ✅ |
| `workModel` | Work mode, see the [reference](#workmodel). 📄 The integration also reads `workMode` | ✅ |
| `batState` | ❓ Battery state, same as in type 2 | ✅ |
| `tempUnit` | ❓ Temperature unit in the app: `0` = °C | ✅ |
| `funcEnable` | Feature bitmask, see the [reference](#funcenable) | ✅ |

> [!TIP]
> `ongridStat` is the most reliable way to tell whether there's grid power. ✅ Verified during real outages: it changes within seconds.

### Power

| Field | Unit | Description | Status |
| --- | --- | --- | :-: |
| `gridInPw` | W | Power from the grid | ✅ |
| `gridOutPw` | W | Power to the grid | ✅ |
| `inGridSidePw` / `outGridSidePw` | W | ❓ The same, measured on the grid side | ✅ |
| `otherLoadPw` | W | 📄 Other household load (needs a CT meter) | ✅ |
| `batInPw` / `batOutPw` | W | Battery charging / discharging | ✅ |
| `swEpsInPw` / `swEpsOutPw` | W | AC outlet | ✅ |
| `pvPw`, `pv1`…`pv4` | W | Solar (without `commState` here) | ✅ |
| `soc` | % | Charge | ✅ |
| `batNum` | count | Number of expansion batteries | ✅ |

📄 Some firmware sends `gridBuyPw` / `gridSellPw` instead of `gridInPw` / `gridOutPw`.

### Limits and planning

| Field | Unit | Description | Status |
| --- | --- | --- | :-: |
| `maxSysOutPw` | W | ❓ Maximum system output: `800` | ✅ |
| `maxSysInPw` | W | ❓ Maximum system input: `2500` | ✅ |
| `maxFeedGrid` | W | 📄 Maximum grid feed-in | ✅ |
| `pvMaxChgPower` | W | ❓ Maximum solar charging | ✅ |
| `chargePlanPw` | W | ❓ Scheduled charging power: `2000` | ✅ |
| `energyPlanPw` | W | ❓ Energy plan power | ✅ |
| `defaultPw` | W | ❓ Default output power | ✅ |
| `isFollowMeterPw` | 0/1 | ❓ Follow the meter when exporting | ✅ |
| `isAutoStandby` | 0/1 | Auto standby allowed. Can be changed with a command | ✅ |
| `standbyPw` | W | ❓ Threshold for entering standby: `25` | ✅ |
| `offGridDown` | 0/1 | ❓ Auto shutdown without grid | ✅ |
| `offGridTime` | min | ❓ Shutdown delay without grid: `120` | ✅ |
| `wps`, `wpc` | — | ❓ Unknown | ✅ |

## Updates and statistics (type 107, 23)

### type 107: partial update 📄

The station may send only the fields that changed, e.g. `soc` and `workMode`. Apply them on top of the last known state. We haven't seen this message.

### type 23: accumulated energy 📄

Energy counters since start-up. **Value × 0.01 = kWh.** If `body.deviceSn` is a sub-device serial, these are plug or meter statistics. If it's missing, the station's own serial or `"system"`, they're the station's.

| Field | Description |
| --- | --- |
| `pvEgy`, `pv1Egy`…`pv4Egy` | Solar generation, total and per input |
| `batChgEgy` / `batDisChgEgy` | Battery charged / discharged |
| `inOngridEgy` / `outOngridEgy` | In / out through the grid port |
| `inEpsEgy` / `outEpsEgy` | In / out through the AC outlet |
| `acOtBatEgy` | AC outlet → battery |
| `acOtOngridEgy` | AC outlet → grid port |
| `pvOtBatEgy` / `pvOtAcEgy` / `pvOtOngridEgy` | Solar → battery / AC outlet / grid port |
| `ongridOtAcLoadEgy` | Grid port → AC outlet |
| `ongridOtBatEgy` | Grid port → battery |
| `batOtAcEgy` / `batOtGridEgy` | Battery → AC outlet / grid port |

## Control commands (type 1)

Topic `action`, `type: 1`, **`eventId: 3`**. `body` always has `"cmd": 5, "rc": 1` plus one or more fields to change:

```json
{
  "type": 1,
  "eventId": 3,
  "messageId": 4821,
  "ts": 1795556302,
  "token": "12345678",
  "body": { "cmd": 5, "rc": 1, "socDischgLimit": 15 }
}
```

| Field | Value | What it does | Status |
| --- | --- | --- | :-: |
| `socDischgLimit` | 5–49 | Discharge limit, % | ✅ |
| `socChgLimit` | 50–100 | Charge limit, % | 📄 |
| `maxOutPw` | 0–2500, step 10 | Grid-port output limit, W | 📄 |
| `swEps` | 0 / 1 | Turn the AC outlet off / on | 📄 |
| `autoStandby` | 0 / 1 / 2 | Auto standby mode | 📄 |
| `isAutoStandby` | 0 / 1 | Allow auto standby | 📄 |
| `reboot` | 1 | Reboot the station | 📄 |

📄 The station may report the limit bounds itself in `minSocChg` / `maxSocChg` and `minSocDischg` / `maxSocDischg`. Ours doesn't send them.

✅ We haven't seen a dedicated reply to a command. To check the result, look for the new value in the next type 2 reply (to request 25), i.e. within a few seconds. A wrong token produces [type 123](#errors-type-123).

> [!CAUTION]
> Commands change the settings of a real station. Validate the ranges, and send `reboot` only on purpose: the station stops powering the outlet for a minute or two.

## Smart plugs and meters (type 100–103)

### List: request type 100 → reply type 101 ✅

```json
{ "type": 100, "eventId": 0, "messageId": 5103, "ts": 1795556393, "token": "12345678", "body": { "devType": 6 } }
```

| `devType` in the request | Returns |
| :-: | --- |
| `6` | Smart plugs: `body.plugs` |
| `2` | CT meters and collectors: `body.cts`, `body.collectors` |

Reply with nothing attached ✅:

```json
{ "type": 101, "eventId": 0, "messageId": 5103, "deviceType": 3, "body": { "cmd": 110, "devType": 6, "plugs": [] } }
```

📄 A type 101 reply is the **complete** list: a device missing from it has been unbound. The lists may be called `plugs`, `plug`, `socket`, `sockets`, `cts` or `ct`.

### Smart plug fields 📄

| Field | Description |
| --- | --- |
| `deviceSn` (or `sn`) | Plug serial number |
| `devType` | `6` |
| `name` | Name from the app |
| `switchSta` / `sysSwitch` | On (0/1) |
| `outPw` / `inPw` | Power, W |
| `totalEgy` | Energy, × 0.01 = kWh |
| `commState` | Link: 1 online |
| `commMode` | 1 local (controllable over MQTT), 2 cloud (app only) |

### CT meter fields 📄

| Field | Description |
| --- | --- |
| `deviceSn` | Serial number |
| `devType` | `2`, `3` or `4`, see the [reference](#sub-device-devtype) |
| `subType` | Meter model, see the [reference](#meter-subtype) |
| `tPhasePw` / `tnPhasePw` | Total power: forward / reverse, W |
| `aPhasePw`, `bPhasePw`, `cPhasePw` | Forward power per phase, W |
| `anPhasePw`, `bnPhasePw`, `cnPhasePw` | Reverse power per phase, W |
| `tPhaseEgy` / `tnPhaseEgy` | Energy: forward / reverse |
| `commState` | Link: 1 online |

📄 Some firmware capitalises the names: `TphasePw`, `AphasePw`, `TnphasePw` and so on.

### Single-device update: type 102 📄

Arrives on `event`. `body` has the sub-device `deviceSn` and the changed fields (e.g. `switchSta`, `outPw`), or `plugs` / `cts` arrays.

### Turning a plug on / off: type 103 📄

```json
{
  "type": 103, "eventId": 0, "messageId": 7310, "ts": 1795556302, "token": "12345678",
  "body": { "deviceSn": "PLUG0EXAMPLE01", "devType": 6, "sysSwitch": 1 }
}
```

Only works for plugs with `commMode: 1`.

## Errors (type 123)

✅ Wrong token: the station replies on `event` (without `deviceType`):

```json
{ "type": 123, "eventId": 0, "messageId": 5105, "ts": 1795556401, "body": { "errorCode": 401 } }
```

We haven't seen any other error codes yet.

## Value reference

### `deviceType`

| Value | Device |
| :-: | --- |
| `1` | 📄 Battery pack |
| `2` | 📄 Meter / collector |
| `3` | ✅ SolarVault 3 ("DIY3" in the integration) |
| `4` | 📄 Meter collector |

### `stat`

| Value | State |
| :-: | --- |
| `0` | ✅ Normal |
| `1` | 📄 Waiting |
| `2` | 📄 Alarm |
| `3` | 📄 Fault |
| `4` | 📄 Standby |
| `5` | 📄 Low power |

### `workModel`

| Value | Mode |
| :-: | --- |
| `0` | 📄 Invalid |
| `1` | 📄 Energy scheduling disabled |
| `2` | 📄 Self-consumption |
| `3` | 📄 Battery priority |
| `4` | ✅ User-defined |
| `5` | 📄 Time of use (TOU) |
| `6` | 📄 Feed-in priority |
| `7` | 📄 Dynamic pricing |

### `funcEnable`

Bitmask, 1 = enabled. 📄 Per the integration:

| Bit | Feature |
| :-: | --- |
| 0 | Aerosol fire suppression |
| 1 | SOC calibration |
| 2 | Low power |
| 3 | SOH calibration |
| 4 | Inverter (PCS) communication diagnosis |
| 5 | 2-hour shutdown |
| 6 | Fault shutdown |
| 7 | Emergency power off (EPO) |
| 8 | 48 V function |
| 9 | Ethernet debug |
| 10 | Energy flow data backfill |
| 11 | Smart plug priority |

✅ Our station sends `2147483648`: bit 31, which the integration doesn't describe.

### Sub-device `devType`

| Value | Device |
| :-: | --- |
| `2`, `3`, `4` | 📄 CT meter and collectors |
| `6` | 📄 Smart plug |

### Meter `subType`

| Value | Model |
| :-: | --- |
| `1` | 📄 Shelly, single phase |
| `2` | 📄 Shelly, three phase |
| `3` | 📄 Shelly 63A |
| `4` | 📄 Eastron, single phase (4002) |
| `5` | 📄 Eastron, three phase (4003) |
| `6` | 📄 Jackery Wireless Smart Meter (US L1/L2, 4007) |
| `7` | 📄 Jackery Smart Meter 3P (UK, 4008) |

## Useful formulas

**Battery power** (+ charging, − discharging), as in the Jackery app:

```
net = batInPw − batOutPw
```

**Load** (what your appliances draw):

```
load = swEpsOutPw + outOngridPw
```

**Capacity** ❓ (SolarVault 3 Pro Max is 2.52 kWh per unit):

```
capacityWh = 2520 × (1 + batNum)
```

**How long the battery will last:**

```
hours = capacityWh × (soc − socDischgLimit) / 100 / batOutPw
```

In practice, smooth `batOutPw` (e.g. a one-minute exponential average). Inverter losses are already included in `batOutPw`. If you compute from `load` instead, add 5–10 % for losses.

## Finding the station on the network

The station doesn't announce itself, but it connects to the broker with a client id equal to its serial number. Mosquitto logs this if `mosquitto.conf` has:

```
log_dest file /opt/homebrew/var/log/mosquitto.log
connection_messages true
```

The log will contain:

```
1795544123: New client connected from 192.168.1.77:52042 as HS2C0EXAMPLE001 (p4, c1, k60, u'jackery').
```

That gives you the serial number and the IP address. You still need the token from the Jackery app.

## Quick check from a terminal

Fill in your own values (`mosquitto-clients` comes with Mosquitto):

```bash
HOST=192.168.1.50 USER=jackery PASS=secret SN=HS2C0EXAMPLE001 TOKEN=12345678

# Terminal 1: listen for replies
mosquitto_sub -h $HOST -u $USER -P $PASS -t "hb/device/$SN/#" -v

# Terminal 2: request station and system state
mosquitto_pub -h $HOST -u $USER -P $PASS -t "hb/device/$SN/action" \
  -m "{\"type\":25,\"eventId\":0,\"messageId\":1,\"ts\":$(date +%s),\"token\":\"$TOKEN\",\"body\":null}"
mosquitto_pub -h $HOST -u $USER -P $PASS -t "hb/device/$SN/action" \
  -m "{\"type\":105,\"eventId\":0,\"messageId\":2,\"ts\":$(date +%s),\"token\":\"$TOKEN\",\"body\":null}"
```

## Known unknowns

What we don't know yet. If you do, [help out](#contributing).

- The meaning of fields marked ❓, especially `ability`, `batState`, `socForceChg`, `wps`, `wpc` and the current limits.
- Whether type 1 commands get a dedicated reply.
- How and when the station sends type 107 and type 23 on its own.
- How to read the firmware version: `softver` doesn't appear in our messages.
- How expansion batteries (`batNum > 0`) and solar versions behave.
- Error codes other than 401.

## Contributing

The most valuable thing is real messages from other stations and firmware:

1. Record a few minutes of traffic: `mosquitto_sub … -t "hb/#" -v > capture.txt`.
2. **Remove** the serial number, token, Wi‑Fi name, IP and MAC addresses.
3. Open an issue with the station model, firmware version and Jackery app version.

## Sources

- The [official Jackery Home Assistant integration](https://github.com/Jackery-Official/jackery) (MIT): type numbers, commands, value references.
- Our own observations of a SolarVault 3 Pro Max AC while building [JackeryBar](https://github.com/Loony-dev/jackerybar).

## License

[MIT](LICENSE). Jackery and SolarVault are trademarks of their respective owners.
