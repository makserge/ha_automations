# Hongtop Projector and MagentaTV One 2nd generation movie mode automation

Control HDMI input, standby, and wake on an Android 10 (API 29) projector using ADB over Wi-Fi.

## Setup

1. Enable Developer options: Settings > About, tap the build number about 7 times.
2. Enable USB debugging and network debugging if available.
3. The projector has a static IP: `192.168.8.111`. Keep it fixed (a static address on the device outside the router's DHCP range, or a DHCP reservation in the router), because Home Assistant and ADB both rely on it.
4. Connect from your PC or Termux. Run this first each time, since the connection drops when the projector reboots or fully powers off:

```
adb connect 192.168.8.111:5555
```

## Verified commands

### Check power state

```
adb shell dumpsys power | grep -i wakefulness
```

Returns `Awake` or `Asleep`.

### Wake

```
adb shell input keyevent 26
```

Key 26 toggles power, so only send it when the state is `Asleep`. Key 224 (KEYCODE_WAKEUP) does not work on this device.

### Enable HDMI-1 (projector must be awake)

```
adb shell am start -n com.android.tv/.SelectInputActivity
sleep 2
adb shell input keyevent 19
adb shell input keyevent 20
adb shell input keyevent 66
```

Opens the input menu, which highlights the currently active input. UP goes to the top item, DOWN moves to HDMI-1, and ENTER selects it. This works from either starting state (tested from the launcher and from HDMI-1 with no signal). Run these as separate commands or as a script, not pasted as one line.

### Disable (leave HDMI and sleep)

```
adb shell input keyevent 3
adb shell input keyevent 26
```

HOME (3) leaves the HDMI input and returns to the launcher. The power key (26) then puts the projector to sleep. Skip the second command if you want it to stay on.

## Key codes used

| Code | Meaning |
|------|---------|
| 3    | HOME    |
| 19   | DPAD_UP |
| 20   | DPAD_DOWN |
| 26   | POWER (toggle) |
| 66   | ENTER   |

## Notes

- The keycodes 243, 244, and 178 do nothing on this firmware. Use the menu method.
- The input menu opens with the currently active input highlighted, so a fixed number of DOWN presses is unreliable. The enable sequence presses UP first to reset to the top.
- Selecting VLC from the input menu was unreliable, so disable uses HOME instead.
- HDMI-1 input ID: `com.allwinner.hdmi/.TVInputSetupService/HW1`.
- If the screen is black on HDMI-1, check the source: `cable_connection_status=0` in `adb shell dumpsys tv_input` means no signal is detected.
- Android sometimes kills `com.android.tv` under low memory. The `am start` line relaunches it.
- The ADB connection does not survive a projector reboot or full power-off.

## Home Assistant

Uses the built-in Android Debug Bridge integration (`androidtv`). Its `adb_command` action runs a shell command directly on the projector, so commands are written without the `adb shell` prefix, and `sleep` and `grep` run on the projector itself.

The projector uses the static IP `192.168.8.111`. The integration is tied to this address, so if it ever changes, the integration must be updated (see below).

### 1. Add the integration

1. Settings > Devices & Services > Add Integration.
2. Search for "Android Debug Bridge" and select it.
3. Enter the projector's IP address `192.168.8.111` and port `5555`. Leave the device class on auto-detect.
4. Submit. On the projector, an "Allow USB debugging?" dialog appears. Tick "Always allow from this computer" and confirm. If the setup fails the first time, run it again after accepting the dialog.

### 2. Find the entity ID

Settings > Devices & Services > Android Debug Bridge > click the device > check the media player entity ID. On this setup it is `media_player.android_projector`. If you need to find it again, search `media_player` in Developer Tools > States, or open the entity and click the gear icon to see the Entity ID field.

### 3. Test one command first

1. Developer Tools > Actions.
2. Switch to YAML mode and paste (with your entity ID):

```yaml
action: androidtv.adb_command
target:
  entity_id: media_player.android_projector
data:
  command: dumpsys power | grep -i wakefulness
```

3. Click Perform action. If it succeeds, ADB works from HA. Then test `input keyevent 3` and watch the projector go to the home screen.

### 4. Create the scripts

Option A, in the UI:

1. Settings > Automations & Scenes > Scripts > Add Script > Create new script.
2. Open the three-dot menu > Edit in YAML.
3. Replace the contents with one of the scripts below and Save.

Option B, in a file: add them to `/config/scripts.yaml` as top-level keys (`projector_on:` and `projector_off:`, without a `script:` wrapper), then Developer Tools > YAML > Reload scripts.

Script 1, HDMI on (wakes if asleep, then selects HDMI-1):

```yaml
alias: Projector HDMI on
mode: single
sequence:
  - action: androidtv.adb_command
    target:
      entity_id: media_player.android_projector
    data:
      command: >-
        dumpsys power | grep -q mWakefulness=Asleep && input keyevent 26 && sleep 3;
        am start -n com.android.tv/.SelectInputActivity;
        sleep 2; input keyevent 19; input keyevent 20; input keyevent 66
```

Script 2, off (leaves HDMI and sleeps if awake):

```yaml
alias: Projector off
mode: single
sequence:
  - action: androidtv.adb_command
    target:
      entity_id: media_player.android_projector
    data:
      command: >-
        input keyevent 3; sleep 1;
        dumpsys power | grep -q mWakefulness=Awake && input keyevent 26
```

On Home Assistant versions older than 2024.8, use `service:` instead of `action:`.

### 5. Run and use them

- Run a script from the Scripts page, or from Developer Tools > Actions with `script.turn_on`. Check the generated entity ID (for example `script.projector_hdmi_on`) on the script's settings page.
- Dashboard button: Edit dashboard > Add card > Button, pick the script entity, set the tap action to run it.
- Automations can call the same scripts as an action.

### Troubleshooting

- Entity shows `unavailable`: ADB connection is down. This happens after a projector reboot or full power-off. HA retries on its own once the projector is back on the network.
- Command does nothing: run it once by hand with `adb shell "<command>"` and compare.
- Menu steps misfire right after waking: raise the `sleep` values.
- Both scripts tested in Home Assistant: "HDMI on" wakes the projector and opens HDMI-1, "off" leaves HDMI and puts it to sleep.

## Automation: projector follows MagentaTV One

The projector wakes and opens HDMI-1 when the MagentaTV One (2nd generation, Android TV) wakes, and goes to sleep when the box has been in standby for 2 minutes. Both directions tested and working.

### 1. Add the box to Home Assistant

1. The box has the fixed IP `192.168.8.158`. Keep it fixed (a static address on the box outside the router's DHCP range, or a DHCP reservation in the router). To check it on the box: Settings > Network & Internet, or Settings > Device Preferences > About > Status.
2. Settings > Devices & Services > Add Integration > **Android TV Remote**. If the box appears under "Discovered", use that card. If discovery or pairing fails ("Failed to connect"), wake the box fully, then add the integration manually and enter its IP (`192.168.8.158`).
3. Enter the pairing code shown on the TV screen.
4. The entity on this setup is `media_player.magentatv_one_2_generation`.

### Requirement: keep the box reachable in standby

Home Assistant can only see and wake the box in standby if the box stays on the network. With the default power setting the box dropped off the network in standby (ping timed out, entity `unavailable`), so neither the wake trigger nor `media_player.turn_on` worked.

Fix: on the MagentaTV One 2nd generation, press Settings on the remote > **Systemeinstellungen** > **Geräteeinstellungen** > **Stromversorgung & Energie** > untick **Stromsparmodus**. After this the entity shows `off` in standby instead of `unavailable`. The trade-off is a higher standby power consumption.

### 2. Check the state behaviour

Developer Tools > States, filter for the entity:

| Box | State |
|-----|-------|
| Awake | `on` |
| Standby | `off` |
| Woken again | `on` |

The automation relies on exactly this behaviour. If a different box reports `unavailable` in standby, the triggers need to include that state.

### 3. Create the automation

Settings > Automations & Scenes > Automations > Create automation > three-dot menu > Edit in YAML:

```yaml
alias: Projector follows MagentaTV One
mode: restart
triggers:
  - trigger: state
    entity_id: media_player.magentatv_one_2_generation
    from: "off"
    not_to:
      - "unavailable"
      - "unknown"
    id: wake
  - trigger: state
    entity_id: media_player.magentatv_one_2_generation
    to: "off"
    not_from:
      - "unavailable"
      - "unknown"
    for: "00:02:00"
    id: sleep
actions:
  - choose:
      - conditions:
          - condition: trigger
            id: wake
        sequence:
          - action: script.turn_on
            target:
              entity_id: script.projector_hdmi_on
          - delay: "00:00:05"
          - action: remote.turn_on
            target:
              entity_id: remote.magentatv_one_2_generation
            data:
              activity: "netflix://"
          - delay: "00:00:08"
          - action: remote.send_command
            target:
              entity_id: remote.magentatv_one_2_generation
            data:
              command: DPAD_CENTER
          - delay: "00:00:08"
          - action: remote.send_command
            target:
              entity_id: remote.magentatv_one_2_generation
            data:
              command:
                - DPAD_DOWN
                - DPAD_DOWN
                - DPAD_CENTER
              delay_secs: 1
          - delay: "00:00:03"
          - action: remote.send_command
            target:
              entity_id: remote.magentatv_one_2_generation
            data:
              command: DPAD_CENTER
      - conditions:
          - condition: trigger
            id: sleep
        sequence:
          - action: script.turn_on
            target:
              entity_id: script.projector_off
```

The script entity IDs (`script.projector_hdmi_on`, `script.projector_off`) must match what Home Assistant generated. Check them in Developer Tools > States by filtering for `script.`.

### 4. How it behaves

- Box wakes: the projector wakes if asleep, then selects HDMI-1 (about 10 to 15 seconds). The box then opens Netflix, picks the profile, and plays the most recently watched title (see below).
- Box enters standby: after 2 minutes the projector goes to the home screen and to sleep. The delay prevents short standby blips from switching the projector off.

### Troubleshooting

- Check the automation's Traces (three-dot menu) to see which branch ran.
- No trace: the entity did not change from `off` to `on`, or the automation is disabled.
- Script error in its Traces: usually the projector ADB entity is `unavailable`.
- Projector wakes but misses HDMI-1: raise the `sleep` values in `projector_hdmi_on` (for example 5 and 3).
- Restarts: the wake trigger only fires when the state leaves `off`, and the sleep trigger ignores changes coming from `unavailable` or `unknown`, so a Home Assistant restart or integration reload does not switch the projector.

### Netflix autostart (part of the wake branch)

After the box wakes, Home Assistant launches Netflix and plays the last-watched title. Tested and working. The `remote` entity is `remote.magentatv_one_2_generation`.

Sequence in the wake branch, with the reason for each step:

1. `script.projector_hdmi_on`: wake the projector and select HDMI-1.
2. Wait 5 s: the box finishes waking.
3. `remote.turn_on` with `activity: "netflix://"`: opens Netflix. The package name `com.netflix.ninja` did nothing on this box, the deep link works.
4. Wait 8 s, then `DPAD_CENTER`: selects the profile on the "Who's watching?" screen. This relies on the wanted profile being highlighted by default.
5. Wait 8 s: the Netflix home screen loads, with the highlight on the banner.
6. `DPAD_DOWN`, `DPAD_DOWN`, `DPAD_CENTER` (1 s apart): the first DOWN goes to the row's category, the second to the last-watched title, OK opens its page.
7. Wait 3 s, then `DPAD_CENTER`: presses Play.

Manual test actions (Developer Tools > Actions, YAML mode):

```yaml
action: remote.turn_on
target:
  entity_id: remote.magentatv_one_2_generation
data:
  activity: netflix://
```

```yaml
action: remote.send_command
target:
  entity_id: remote.magentatv_one_2_generation
data:
  command: DPAD_CENTER
```

Troubleshooting:

- The delays are timings that worked once. If a step fires too early, raise the delay before it. Find the first screen that goes wrong: profile screen (raise the 8 s after launch), banner (raise the second 8 s), details page (raise the 3 s).
- The key sequence depends on Netflix's layout. If Netflix reorders rows or changes the home screen, the DOWN count may need to change. Re-test by hand with `DPAD_DOWN` and watch where the highlight goes.
- If the profile highlighted by default is not the wanted one, add a DPAD move before the profile OK.
- Netflix does not resume from a link. Only a specific title can be opened directly with `netflix://title/<id>`, which is fixed rather than the last watched.

## Automation: cinema mode via MQTT

Topic: `ha_automations/cinema_mode`

| Payload | Action |
|---------|--------|
| `1` | All lights off, ambient light on at 30%, projector on, MagentaTV One on |
| `0` | All lights off, ambient light off, projector off, MagentaTV One off |

Entities used:

- Lights: `switch.hall_ceiling_light`, `switch.wall_bra`, `light.ledstriphalldimmer_1` (Led Strip Hall, also the ambient light)
- Projector: scripts `script.projector_hdmi_on` and `script.projector_off`
- Box: `media_player.magentatv_one_2_generation`

```yaml
alias: Cinema mode (MQTT)
mode: restart
triggers:
  - trigger: mqtt
    topic: ha_automations/cinema_mode
    payload: "1"
    id: cinema_on
  - trigger: mqtt
    topic: ha_automations/cinema_mode
    payload: "0"
    id: cinema_off
actions:
  - action: homeassistant.turn_off
    target:
      entity_id:
        - switch.hall_ceiling_light
        - switch.wall_bra
        - light.ledstriphalldimmer_1
  - choose:
      - conditions:
          - condition: trigger
            id: cinema_on
        sequence:
          - action: light.turn_on
            target:
              entity_id: light.ledstriphalldimmer_1
            data:
              brightness_pct: 30
          - if:
              - condition: state
                entity_id: media_player.magentatv_one_2_generation
                state: "off"
            then:
              - action: media_player.turn_on
                target:
                  entity_id: media_player.magentatv_one_2_generation
            else:
              - action: script.turn_on
                target:
                  entity_id: script.projector_hdmi_on
      - conditions:
          - condition: trigger
            id: cinema_off
        sequence:
          - action: script.turn_on
            target:
              entity_id: script.projector_off
          - action: media_player.turn_off
            target:
              entity_id: media_player.magentatv_one_2_generation
```

How it behaves:

- Payload `1` with the box in standby: the box is turned on, which starts the "Projector follows MagentaTV One" automation (projector wakes, HDMI-1, Netflix).
- Payload `1` with the box already on: there is no wake event, so the projector script runs directly. Netflix is not restarted in this case.
- Payload `0`: lights and ambient off, projector off, box off. The 2-minute standby trigger in the other automation also fires later and does no harm, because the projector-off script only presses power when the projector is awake.
- Publish without the retain flag. A retained message would run the automation again after every Home Assistant restart.

Testing: Developer Tools > MQTT > Publish a packet, topic `ha_automations/cinema_mode`, payload `1` or `0`. Tested end to end with both payloads: `1` switches the lights off, sets the ambient light to 30%, wakes the box, and the projector and Netflix start; `0` switches the lights and ambient light off, and puts the projector and the box to sleep. This relies on the box staying reachable in standby (see the Stromsparmodus requirement above).

## If the MagentaTV One's IP changes

The box uses the fixed IP `192.168.8.158`. The address is stored only in the Android TV Remote integration in Home Assistant. The automations refer to the entities `media_player.magentatv_one_2_generation` and `remote.magentatv_one_2_generation`, so the YAML does not need editing.

If the address changes: Settings > Devices & Services > Android TV Remote > three-dot menu > Reconfigure if it is offered. Otherwise delete the entry and add it again with the new IP, and enter the pairing code shown on the TV screen. After re-adding, check the entity IDs in Developer Tools > States (filter for `media_player` and `remote`). If they differ, update them in the automations.

### Dashboard buttons for cinema mode

The automation is triggered by the MQTT message, so the dashboard buttons publish that message with the `mqtt.publish` action (file `ha/dashboard_cinema_card.yaml`).

1. Open the dashboard and click the pencil icon (Edit dashboard).
2. Add card > scroll down to **Manual**.
3. Paste the card YAML and save.

```yaml
type: horizontal-stack
cards:
  - type: button
    name: Cinema on
    icon: mdi:movie-open
    tap_action:
      action: perform-action
      perform_action: mqtt.publish
      data:
        topic: ha_automations/cinema_mode
        payload: "1"
        retain: false
  - type: button
    name: Cinema off
    icon: mdi:movie-off
    tap_action:
      action: perform-action
      perform_action: mqtt.publish
      data:
        topic: ha_automations/cinema_mode
        payload: "0"
        retain: false
```

Do not call the automation directly (`automation.trigger`) from a button. It would run without the MQTT payload and the trigger ID, so neither branch would be chosen. On Home Assistant versions older than 2024.8, use `action: call-service` with `service: mqtt.publish` instead of `perform-action`.

## If the projector's IP changes

The IP address appears only in two places: the `adb connect` command and the Android Debug Bridge integration in Home Assistant. The scripts and automations refer to entities (`media_player.android_projector`), not to the address, so they do not need editing.

To update the integration after an IP change: Settings > Devices & Services > Android Debug Bridge > three-dot menu > Reconfigure (or delete the entry and add it again), then enter the new IP and port 5555. The Home Assistant ADB key is already authorised on the projector, so the "Allow USB debugging?" dialog should not appear again. If the entry is re-created, the entity ID may change, so check it in Developer Tools > States (filter for `media_player`) and update `media_player.android_projector` in the scripts if it differs.

