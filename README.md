# ZHA Quirk for MOES BHT-002-GCLZB / ZHT-002 Thermostat (`_TZE204_xalsoe3m`)

A custom Home Assistant ZHA quirk that adds support for the MOES Tuya Zigbee
thermostat with the manufacturer ID `_TZE204_xalsoe3m`. This newer hardware
revision uses a different Tuya datapoint layout than the older `_TZE200_*` /
`_TZE204_aoclfnxz` models, so the existing Avatto/Beok quirks don't work for it
and the device shows up in ZHA with only LQI/RSSI entities.

This quirk fills that gap so the thermostat can be used as a climate entity in
Home Assistant.

## Supported devices

This quirk targets devices reporting as:

- **Model:** `TS0601`
- **Manufacturer:** `_TZE204_xalsoe3m`

Sold under various names, including:

- MOES **BHT-002-GCLZB** (Zigbee boiler thermostat, white)
- MOES **ZHT-002-GA-WH-MS** / **ZHT-002-GC-WH-MS**

The product is a wall-mounted Zigbee thermostat for water/gas boilers
(not a TRV). If your device reports a different manufacturer ID, this quirk
won't match it.

## Status

**Partial support — core control works, advanced features may not.**

Working:

- Current room temperature (read)
- Target temperature / setpoint (read and write from HA)
- System mode on/off
- Running state (heating / idle)
- Quirk loads cleanly and ZHA recognizes the device as a climate entity

Not yet verified or only partially working:

- Schedule / preset mode switching
- Child lock toggling from HA
- Temperature calibration writes
- Deadzone temperature writes
- Sensor selection (internal / external / both)
- Min / max temperature limits

If you need full feature parity, **Zigbee2MQTT has a more complete
implementation for this device** (see references below). This quirk exists for
people who want to stay on ZHA.

## Installation

1. Copy `ts0601_thermostat_avatto.py` into your Home Assistant config under:

   ```
   /config/custom_zha_quirks/ts0601_thermostat_avatto.py
   ```

2. In `configuration.yaml`, point ZHA at the custom quirks directory:

   ```yaml
   zha:
     custom_quirks_path: /config/custom_zha_quirks/
   ```

3. Restart Home Assistant fully (not a YAML reload — ZHA only loads quirks on
   a full restart).

4. **Remove the existing thermostat from ZHA** and pair it again. A simple
   "Reconfigure" usually isn't enough because the device gets matched against
   a different device class with a new endpoint layout, and the cluster
   configuration needs to be rebuilt from scratch.

After re-pairing, check the device info page in Home Assistant. It should now
show:

```
Quirk: ts0601_thermostat_avatto.MoesXalsoe3m
```

(or `MoesXalsoe3mNoGP` if your device has no GreenPower endpoint).

## How it works

The MOES `_TZE204_xalsoe3m` uses these Tuya datapoints, which were
reverse-engineered for Zigbee2MQTT by `SSG1111`:

| DP | Attribute             | Meaning                                          |
|----|-----------------------|--------------------------------------------------|
| 1  | `0x0101`              | system_mode (on/off)                             |
| 2  | `0x0202`              | preset (0=schedule, 1=manual)                    |
| 16 | `0x0210`              | local_temperature (value ÷ 10 = °C)              |
| 19 | `0x0213`              | local_temperature_calibration                    |
| 32 | `0x0220`              | sensor (IN / OU / AL)                            |
| 39 | `0x0227`              | child_lock                                       |
| 47 | `0x042F`              | running_state (0=idle, 1=heat)                   |
| 50 | `0x0232`              | current_heating_setpoint (raw integer °C)        |
| 102| `0x0266`              | deadzone_temperature                             |

Note the asymmetric scaling: the device reports `local_temperature` in
tenths of a degree, but expects (and returns) `current_heating_setpoint` as a
raw integer in whole degrees. The quirk handles this conversion to and from
the centidegree format that ZHA's climate cluster uses internally.

## Troubleshooting

**Device shows up but climate entity stays "Unknown" after pairing**

The thermostat only starts reporting datapoints once it's powered on. Press
the power button on the physical device until the display lights up and shows
a setpoint.

**Setpoint changes in HA don't reach the device, or values revert**

Enable ZHA debug logging in `configuration.yaml`:

```yaml
logger:
  default: warning
  logs:
    zigpy: debug
    zhaquirks: debug
    zhaquirks.tuya: debug
```

Restart, change the setpoint, then check the logs for lines like:

```
Mapping standard occupied_heating_setpoint (0x0012) with value 2700 to custom {562: 27}
Received value [0, 0, 0, 27] for attribute 0x0232
```

The `to custom {562: 27}` part shows what's being written to the device
(`562` = `0x0232` = setpoint datapoint). If the value or datapoint looks wrong,
open an issue with the log excerpt.

**Quirk doesn't load at all**

Check Home Assistant logs for Python syntax or import errors under the
`zhaquirks` logger. Make sure you copied the whole file and the path in
`configuration.yaml` matches exactly.

## Credits

- Base quirk: [jacekk015/zha_quirks](https://github.com/jacekk015/zha_quirks)
  (`ts0601_thermostat_avatto.py`)
- Datapoint reverse-engineering for `_TZE204_xalsoe3m`:
  [SSG1111 in Koenkk/zigbee2mqtt#25774](https://github.com/Koenkk/zigbee2mqtt/discussions/25774)
- Related ZHA issue:
  [zigpy/zha-device-handlers#4507](https://github.com/zigpy/zha-device-handlers/issues/4507)

## License

Same license as the upstream `jacekk015/zha_quirks` repository
(check there for the exact terms).

## Contributing

PRs and issue reports welcome, especially for the unverified features
listed above. If you have the same device and got something working that
this quirk doesn't, please share.
