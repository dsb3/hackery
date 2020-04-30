# MQTT to HA Pulse Meters

I have this meter installed on one of my water pipes for which I want to measure usage -> https://www.adafruit.com/product/828 (the product docs further identify it as a "YF-S201 sensor").

Elsewhere in my repos is the code that drive the ESP8266/ESP32 with it attached, but
here I'll simply capture the configuration elements that make it work with Home
Assistant.


# Configuration
  
As usual, my main configuration.yaml just has fragments to let me include each configuration block in it's own file for easier management.
````yaml
...
sensor: !include sensor.yaml
binary_sensor: !include binary_sensor.yaml
utility_meter: !include utility_meter.yaml
...
````


## Sidebar : polling vs mqtt

My ESP code includes a web server that lets you poll it for updates instead of listening to MQTT topics.  This is less featureful (you will simply receive periodic updates, rather than receive updates timed around the start and end of water flow) and also has additional network limitations, along with the inability to use a public MQTT broker if one is preferred.

sensor.yaml
````yaml
- platform: rest
  resource: http://192.168.x.y/flow.json
  name: waterflow_l
  value_template: '{{ ( value_json.flowpulses | float / 450.0 ) | round(2) }}'
  unit_of_measurement: "L"
  scan_interval: 15
  force_update: true
````

The interesting element here is "force_update".  If this is not set, once water stops flowing (the pulses no longer increment) the sensor will not update with a duplicate value to the one prior, and the binary_sensor (shown later) that detects current water flow will never turn off.

Enough about this.  Getting data over MQTT is nicer for my usage.


## MQTT Configuration

**sensor.yaml:**

(don't include both the JSON and the MQTT sensors; or if you do, be sure to name them differently)

**sensor.yaml:**
```yaml
...
# read raw pulse data via MQTT updates
- platform: mqtt
  name: waterflow_pulse
  unit_of_measurement: "pulses"
  force_update: true
  state_topic: "ha/sensor/BCDDC2aabbcc/waterflow/state"

- platform: template
  sensors:
    waterflow_liter:
      unit_of_measurement: "L"
      availability_template: '{{ states("sensor.waterflow_pulse") != "unknown" }}'
      value_template: '{{ (states("sensor.waterflow_pulse") | float / 450.0 ) | round(2)  }}'

...
```


**binary_sensor.yaml:**

Create a "trend" sensor.  This simply reports dry/wet for whether water is flowing or not.  Because we have updates sent immediately after water starts flowing, and continues immediately following the water stopping, this is responsive to within a few seconds of the actual water usage.

```yaml
...
- platform: trend
    sensors:
      garage_water_flowing:
      device_class: moisture
      entity_id: sensor.waterflow_pulse
      min_gradient: 0.00001    # squelch e-18 type rounding errors
...
```


**utility_meter.yaml:**

Create two utility meters - daily and monthly.  These make for clear output and manage water usage across device restarts (which also causes waterflow\_l to reset to zero).

To-do: the utility meter integration claims to have a "reset" service call but I've not yet gotten it to work.

```yaml
...
garage_water_daily:
  source: sensor.waterflow_l
  cycle: daily

garage_water_monthly:
  source: sensor.waterflow_l
  cycle: monthly
...
```                           


# Data loss

There's always the possibility of data loss with this type of meter.  If you can't accept that, you need a different type entirely.  But, for this, we can minimize it.

Use pulses, not per period readings.

Example -- 10 liters of water flows for each of three hours.

If you get readings after each of the three hours, you have a full data set for 30l total.

If you miss the middle reading, and are using "per period" water usage, you think that only 20l was used.

If you miss the middle reading, and are using "incremental pulse" water usage, you know a total of 30l was used, and simply don't know how it was allocated between hours #2 and #3.

Using the availability\_template we can tell if HA itself restarts and avoid mis-understanding the loss of data to be "0" and thus double count the pulses when they next are sent.

Using the "retain: true" on the MQTT messages sent from the embedded device will further prevent mistakenly seeing the pulse count drop to "0" and immediately back to it's previous value.


# Future

There are a few potential future enhancements to this integration.

1/ Alert if water flows and the corresponding motion sensor hasn't triggered.  If water flows

2/ Alert on slow leaks.  Minor, but continued, low volume flow that cannot be attributed to sensor flutter.

3/ Alert on extended flow.  If water stays on more than X minutes, we should send an alert.

4/ Alert on high flow.  We know the maximum flow if the faucet is wide open.  If water ever flows through the sensor at a higher volume we can probably conclude the pipe has burst and an alert would be nice.


