# What is this?
A guide to help you run ESPHome (or whatever you want) instead of the crappy factory Cheinese spy firmware on the Xiaomi Mi Smart Space Heater S (and most likely other, similar models).

# Prerequisites
* the device itself
* tools to disassemble. You'll need a special "triangular head" security bit (my iFixit kit has it all) and some rather long and narrow philips.
* USB - TTL UART adapter
* soldering iron & skills
* know-how about physical ttl/uart flashing
* This model has a thermistor as its temperature sensor which I couldn't find data about. I calibrated it but your mileage may vary so be ready to either replace a thermistor with a known-value one or know how to calibrate one.

# Disclaimer
This current implementation renders the physical buttons lifeless. It could possibly be resolved but I simply didn't and don't care enough. Feel free to fix and PR, should you feel the motivation for it.

# Flashing
Disassemble and find the logic board with an ESP32 on it. Solder the usual pins and flash ESPHome onto it so that you can OTA flash from now on. You may as well reassemble and close everything up at this point.
Please note: this ESP32 has an external antenna and will pretty much fail to connect with it being disconnected. Don't be scared if it cannot connect until you reassembled everything!

# ESPHome setup
The following configuration works great and creates a Thermostat component as well. I've found that you can cut all the Xiomi proprietary protocol bullshit altogether and just control the relays and read the themistor as an analog value. Feel free to experiment and switch the relays and thus heating elements separately. 

```
esphome:
  name: miheater
  friendly_name: MiHeater

esp32:
  board: esp32dev
  framework:
    type: esp-idf
    sdkconfig_options:
      CONFIG_FREERTOS_UNICORE: y
    advanced:
      ignore_efuse_mac_crc: true

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "YOUR_KEY_HERE"

ota:
  - platform: esphome
    password: "YOUR_PASSWORD_HERE"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Xiaomi Mi Fallback AP"
    password: "AP PASSW HERE"

captive_portal:

web_server:

# Thermistor
# -1C      14.1 KOhm         272.15K        14100 Ohm
# 27.2C    4.65 KOhm         300.35K        4650  Ohm
# 59C      0.79 KOhm				332.15K				  790   Ohm
# ln(14100/790) / (1/272.15 - 1/332.15) = 4341
sensor:
  - platform: ntc
    sensor: resistance_sensor
    calibration:
      b_constant: 4341
      reference_temperature: 27.2°C
      reference_resistance: 4.65kOhm
    name: "NTC Temperature"
    id: temperature_sensor

  # Example source sensors:
  - platform: resistance
    id: resistance_sensor
    sensor: source_sensor
    configuration: DOWNSTREAM
    resistor: 5.6kOhm
    name: "Resistance Sensor"
  - platform: adc
    id: source_sensor
    name: "ADC sensor"
    pin: GPIO36
    attenuation: auto
    update_interval: 5s

# Heating rod relays
switch:
  - platform: gpio
    pin: GPIO4
    name: "Heating relay 1"
    id: element1_relay
  - platform: gpio
    pin: GPIO2
    name: "Heating relay 2"
    id: element2_relay

# Climate abstraction
climate:
  - platform: thermostat
    name: "Thermostat"
    sensor: temperature_sensor
    min_heating_off_time: 5s
    min_heating_run_time: 5s
    min_idle_time: 5s
    heat_action: 
      - switch.turn_on: element1_relay
      - switch.turn_on: element2_relay
    idle_action: 
      - switch.turn_off: element1_relay
      - switch.turn_off: element2_relay
    heat_deadband: 0.5°C
    heat_overrun: 0.5°C
    visual:
      min_temperature: 15
      max_temperature: 30
      temperature_step: 1
```

        
        
    
      
      
