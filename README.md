# ha-venetian-blind
Home Assistant Venetian Blind Cover Component

All credits go to the author of this component **Václav Chaloupka** - [bruxy70](https://github.com/bruxy70)

This is for ESP microcontrollers to control time based venetian blinds - controlling the action (open/close/stop), position (0-100) and the tilt (0-100).

For the position, 0 is closed, 100 is open - consistent with standard blinds and other 'things' that open and close (like valves) in Home Assistant and in general.
Intuitively I'd do it the other way around (so that it would represent how much of the cover is out). I had number of discussions with HA developers, even tried to reverse it in the custom cover controller. But at the end, I agreed that despite all concerns this does make most sense, and trying to hack it causes more problems than benefits.
If you do not like this from UI perspective, use a custom card and show it differently on front-end. But leave it standard at the back.

There are two versions of the controller: current - an ESPHome custom controller. And the legacy - based on Arduino code, using MQTT.

## ESPHome controller

This uses a custom cover controller. Similar to the time based cover, but with added tilt control. To load the custom controller, add the following code to your ESPHome device configuration:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/bruxy70/Venetian-Blinds-Control
      ref: master
    components: [venetian_blinds]
```

Then you can configure the blinds. For the parameters and use refer to the [Time Based Cover](https://esphome.io/components/cover/time_based.html), with one additional parameter for `tilt_duration`.

Configuration example:

```yaml
cover:
  - platform: venetian_blinds
    name: "${cover1_name}"
    id: cover1
    open_action:
      - switch.turn_on: relay1
    open_duration: 51770ms
    close_action:
      - switch.turn_on: relay2
    close_duration: 51200ms
    tilt_duration: 1650ms
    stop_action:
      - switch.turn_off: relay1
      - switch.turn_off: relay2
```

To control the blinds by the wall switch, configure a binary switch with action to open and close them.
```yaml
  - id: key1
    platform: gpio
    pin:
      number: ${key1_gpio}
      mode: INPUT_PULLUP
      inverted: True
    on_press:
      then:
        - cover.open: cover1
    on_release:
      then:
        - cover.stop: cover1
  - id: key2
    platform: gpio
    pin:
      number: ${key2_gpio}
      mode: INPUT_PULLUP
      inverted: True
    on_press:
      then:
        - cover.close: cover1
    on_release:
      then:
        - cover.stop: cover1
```