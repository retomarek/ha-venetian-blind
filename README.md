# ha-venetian-blind
Home Assistant Venetian Blind Cover Component

All credits go to the author of the original component on which this one is based on: **Václav Chaloupka** - [bruxy70](https://github.com/bruxy70)

This solution here differs from his in the following points:
- tilt percentage was somehow inverted, now 100 means fully closed and 0 fully horizontal
- a minimal direction change delay of 500ms is implemented to avoid a short circuit
- the press behaviour of the button is: short press -> step of about 100ms, long press -> fully move up/down

## ESPHome controller for shelly 2.5 PM

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/retomarek/ha-venetian-blind
      ref: master
    components: [venetian_blinds]
```

Then you can configure the blinds. For the parameters and use refer to the [Time Based Cover](https://esphome.io/components/cover/time_based.html), with one additional parameter for `tilt_duration`.

Configuration example:

```ESPHome yaml
substitutions:
  node_name: kidsroom-shelly-cover
  device_name: Kidsroom Shelly Cover
  cover_name: kidsroom_test
  ip: 192.168.xx.xx
  channel_1: Open relay
  channel_2: Close relay

  ssid: !secret wifi_ssid
  password: !secret wifi_password

  max_power: "2000.0"
  max_temp: "70.0"

esphome:
  name: $node_name
  comment: $device_name

esp8266:
  board: esp01_1m

wifi:
  ssid: ${ssid}
  password: ${password}
  power_save_mode: HIGH # for ESP8266 LOW/HIGH are mixed up, esphome/issues/issues/1532
  manual_ip:
    static_ip: 192.168.178.65
    gateway: 192.168.178.1
    subnet: 255.255.255.0

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: ${device_name}
    password: ${password}

captive_portal:

# Enable logging
logger:
  level: DEBUG

# Enable Home Assistant API
api:
  password: ${password}

ota:
  password: ${password}

web_server:
  port: 80

external_components:
#  - source:
#      type: local
#      path: /config/custom_components/
#    components: [venetian_blinds]
#    refresh: 1min
  - source:
      type: git
      url: https://github.com/retomarek/ha-venetian-blind
      ref: master
    components: [venetian_blinds]
time:
  - platform: sntp
    id: my_time

i2c:
  sda: GPIO12
  scl: GPIO14

sensor:
  - platform: ade7953
    irq_pin: GPIO16 # Prevent overheating by setting this
    voltage:
      name: ${device_name} voltage
    # On the Shelly 2.5 channels are mixed ch1=B ch2=A
    current_a:
      name: ${channel_2} current
      internal: true
    current_b:
      name: ${channel_1} current
      internal: true
    active_power_a:
      name: ${channel_2} power
      id: power_channel_2
      # active_power_a is normal, so don't multiply by -1
      on_value_range:
        - above: ${max_power}
          then:
            - switch.turn_off: close_relay
            - homeassistant.service:
                service: persistent_notification.create
                data:
                  title: Message from ${device_name}
                data_template:
                  message: Switch turned off because power exceeded ${max_power}W
    active_power_b:
      name: ${channel_1} power
      id: power_channel_1
      # active_power_b is inverted, so multiply by -1
      filters:
        - multiply: -1
      on_value_range:
        - above: ${max_power}
          then:
            - switch.turn_off: open_relay
            - homeassistant.service:
                service: persistent_notification.create
                data:
                  title: Message from ${device_name}
                data_template:
                  message: Switch turned off because power exceeded ${max_power}W
    update_interval: 30s

  - platform: total_daily_energy
    name: ${channel_1} energy
    power_id: power_channel_1
    filters:
      # Multiplication factor from W to kWh is 0.001
      - multiply: 0.001
    unit_of_measurement: kWh

  - platform: total_daily_energy
    name: ${channel_2} energy
    power_id: power_channel_2
    filters:
      # Multiplication factor from W to kWh is 0.001
      - multiply: 0.001
    unit_of_measurement: kWh

  # NTC Temperature
  - platform: ntc
    sensor: temp_resistance_reading
    name: ${device_name} temperature
    unit_of_measurement: "°C"
    accuracy_decimals: 1
    icon: "mdi:thermometer"
    calibration:
      b_constant: 3350
      reference_resistance: 10kOhm
      reference_temperature: 298.15K
    on_value_range:
      - above: ${max_temp}
        then:
          - switch.turn_off: open_relay
          - switch.turn_off: close_relay
          - homeassistant.service:
              service: persistent_notification.create
              data:
                title: Message from ${device_name}
              data_template:
                message: Switch turned off because temperature exceeded ${max_temp}°C
  - platform: resistance
    id: temp_resistance_reading
    sensor: temp_analog_reading
    configuration: DOWNSTREAM
    resistor: 32kOhm
  - platform: adc
    id: temp_analog_reading
    pin: A0

status_led:
  pin:
    number: GPIO0
    inverted: yes

switch:
  - platform: gpio
    id: open_relay
    name: ${channel_1}
    pin: GPIO4
    icon: "mdi:blinds-open"
    restore_mode: RESTORE_DEFAULT_OFF
    interlock: &interlock_group [open_relay, close_relay]
    interlock_wait_time: 500ms
  - platform: gpio
    id: close_relay
    name: ${channel_2}
    pin: GPIO15
    icon: "mdi:blinds"
    restore_mode: RESTORE_DEFAULT_OFF
    interlock: *interlock_group
    interlock_wait_time: 500ms

binary_sensor:
  - platform: gpio
    pin:
      number: GPIO13
    name: "${channel_1} input"
    on_multi_click:
      #Single Click
      - timing:
          - ON for at most 0.39s
          - OFF for at least 0.1s
        then:
          - cover.open: cover1
          - delay: 0.1s
          - cover.stop: cover1
      #Hold
      - timing:
          - ON for at least 0.4s
        then:
          - cover.open: cover1
  - platform: gpio
    pin:
      number: GPIO5
    name: "${channel_2} input"
    on_multi_click:
      #Single Click
      - timing:
          - ON for at most 0.39s
          - OFF for at least 0.1s
        then:
          - cover.close: cover1
          - delay: 0.15s
          - cover.stop: cover1
      #Hold
      - timing:
          - ON for at least 0.4s
        then:
          - cover.close: cover1

cover:
  - platform: venetian_blinds
    name: "${cover_name}"
    id: cover1
    open_action:
      - switch.turn_on: open_relay
    open_duration: 56000ms
    close_action:
      - switch.turn_on: close_relay
    close_duration: 56000ms
    tilt_duration: 1650ms
    stop_action:
      - switch.turn_off: open_relay
      - switch.turn_off: close_relay


```
