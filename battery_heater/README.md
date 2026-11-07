# Battery heater automation

**Battery:** Marstek B2500 (LFP)
**Temperature sensor:** `sensor.b2500lite_b1_temp_min` (minimum cell temperature)
**Heater relay:** `switch.pmrelay4_1` (16 W heater)
**Automation mode:** `single`

## Temperature control
- Heater turns **on** when the temperature falls **below 17 °C**
- Heater turns **off** when the temperature rises **above 20 °C**
- Triggers fire only when a threshold is crossed (3 °C hysteresis band, no flickering)

## Safety features
- **Overheat protection:** above 30 °C the heater is switched off and a notification is sent
- **Sensor-lost failsafe:** if the sensor is `unavailable` or `unknown` for 15 minutes, the heater is switched off
- **Maximum run time:** after 12 hours of continuous heating, the heater is switched off for 1 hour and a notification is sent
- **Automatic resume:** 1 hour after the heater has been switched off, it turns back on if the temperature is still below 17 °C
- **Heat-up check:** if the heater has been on but the temperature has stayed below 17 °C for 2 hours, a notification is sent

## Home Assistant restart handling
- On startup, the heater is switched **on** if the temperature is below 17 °C
- On startup, the heater is switched **off** if the temperature is above 20 °C

## Notifications
- Heater has run for more than 12 hours (heater is also paused for 1 hour)
- Temperature is above 30 °C (heater is also switched off)
- Heater is on but the temperature is still below 17 °C after 2 hours
- Delivered as Home Assistant persistent notifications (visible in the HA interface only)