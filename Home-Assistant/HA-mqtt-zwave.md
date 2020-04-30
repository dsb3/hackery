
# MQTT to ZWave Integration  
  
My core Home Assistant installation was made long before I even thought about adding MQTT to the mix. The latest use case to add MQTT is for one-off action buttons to enable others in the house to control certain lights.  
  
Although my first use case is for ZWave light-switches, the exact same configuration will work for ZWave switches, hue lights, and others. As long as the entity itself can accept a `toggle` service call, it can work with this simple pair of automations.  
  
Until I did this my only MQTT integrations were for simple state updates, so understanding the separation of control and state MQTT topics was a little bit of a learning curve.  
  
## Create MQTT topics  
  
For my example I have a ZWave light switch controlling exterior lights on the shed.  

configuration.yaml just has fragments to let me include each configuration block in it's own file for easier management.
````yaml
...
switch: !include switch.yaml
...
````

switch.yaml
````yaml
- platform: mqtt
  name: "MQTT Shed Door Lightswitch"
  state_topic: "ha/switch/shed/doorlight/state"
  command_topic: "ha/switch/shed/doorlight/command"
````


## Create Automation #1 - control  
  
Automation #1 - the control topic - was the hardest one to understand.  

configuration.yaml has another include fragment for automations
````yaml
...
automation: !include automations.yaml
...
````

automations.yaml
````yaml
- id: '1579164640'
  alias: MQTT - Toggle switch on command
  description: ''
  trigger:
  - platform: mqtt
    topic: ha/switch/shed/doorlight/command
  condition: []
  action:
  - data:
      entity_id:
      - switch.shed_door_lights
    service: switch.toggle
````

  
I simply wasn't able to get `trigger.payload` to work at all, and decided that instead of trying to fix it, there was benefit in ignoring it.  
  
  
## Create Automation #2 - state  
  
Automation #2 - the state topic - was the easiest.  
This simply watches for any state change in our switch entity, capitalizes the entire string, and sends it as an MQTT update to the state topic  

automations.yaml
````yaml
- id: '1579164650'
  alias: MQTT - Update switch state
  description: ''
  trigger:
  - entity_id: switch.shed_door_lights
    platform: state
  condition: []
  action:
  - data:
      payload_template: '{{ states("switch.shed_door_lights") | upper }}'
      topic: ha/switch/shed/doorlight/state
    service: mqtt.publish

````

  
## End to End  
  
(Automation #1)  
**Controller:** understands state is currently "OFF". User hits the button, and sends an "ON" message to the control topic.  
  
**HA:** receives the control topic, and sends a `switch.toggle` service call to the switch in question.  
  
[[ either as a result of that `switch.toggle`, or something else entirely like a manual flip of the switch or some other action that causes it to update ]]  
  
(Automation #2)  
**HA:** receives notification that the state of the switch changed. Sends the current state to the state topic.  
  
**Controller:** receives the state topic, updates it's internal state of the switch.  
  
  
### Self Correcting States  
  
There's an edge case at play here, whereby if the controller and HA get out of sync they might get stuck. The controller mistakenly thinks the switch is off, and so sends an "ON" command to turn it on. However, HA knows the switch is on already, so when it gets the "ON" command and calls `switch.turn_on` **nothing happens**. The user presses the button again, sends another "ON" command, and again nothing happens.  
  
To solve this, we use `switch.toggle`. Regardless of what control topic is sent - ON, OFF, or anything else - we toggle the switch. As a result of toggling the switch, we send a state update.  
  
Instead of getting stuck in a loop where we can't update the switch, we either (if controller and HA agree on switch state) operate exactly as expected or (if they disagree) the first control message does the opposite of what was intended, but is immediately followed by a true state update.  
  
  
  
  
  
  
## Testing

To test the automation I reworked it (using `light.toggle`) to control a light in my office to more easily watch state changes.  

Adding the MQTT switch to lovelace lets me toggle the switch manually.

Sending MQTT messages (mosquitto_pub) also lets me toggle.

Finally, adding a toggle button to MQTT Dashboard ([link](https://play.google.com/store/apps/details?id=com.app.vetru.mqttdashboard)) lets me toggle and see updated status of the target switch.

### Bugs  

I noticed that the light will sometimes toggle automatically when the automation was saved. I suspect this is due to the mqtt `retain` flag being set on prior message, causing a state change to appear to happen even though the before and after states are identical.

The edge case of the first control message doing the opposite action as intended could be fixed with significant added complexity.  Programatically this is easy -- look to see if the control message already matches the current state and if so send a fresh state update.  The implementation would vastly complexify things, and as it stands having the whole functionality present in two small automations is very much preferable.


### Future thoughts

If the zwave switch is slow to respond, a user may continually bash on their control button and send many toggle commands.

If the automation "to" state can contain a template which is "not what we are" then toggle is always appropriate.

Extending to THREE automations total, we could have:

control message received that matches current state -> send state update
control message received that does not match current state -> send toggle  (This would also handle the situation of unknown -> ON for example)

entity state changed -> send state update to mqtt


