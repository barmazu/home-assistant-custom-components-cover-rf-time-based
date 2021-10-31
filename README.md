# Cover Time Based Component trigger script (RF) version
Cover Time Based Component for your [Home-Assistant](http://www.home-assistant.io) based on [davidramosweb's Cover Time Based Component](https://github.com/davidramosweb/home-assistant-custom-components-cover-time-based), modified for native cover entities, covers triggered by RF commands, or any other unidirectional methods.

With this component you can add a time-based cover. You either have to set triggering scripts to open, close and stop the cover or you can use an existing cover entity provided by another integration which does not have timing or status feedback. Position is calculated based on the fraction of time spent by the cover travelling up or down. You can set position from within Home Assistant using service calls. When you use this component, you can forget about the cover's original remote controllers or switches, because there's no feedback from the cover about its real state, state is assumed based on the last command sent from Home Assistant. There's also a custom service available where you can update the real state of the cover based on external sensors if you want to.

You can adapt it to your requirements, actually any cover system could be used which uses 3 triggers: up, stop, down. The idea is to embed your triggers into scripts which can be hooked into this component via config. For example, you can use RF-bridge or dual-gang switch running Tasmota or ESPHome firmware integrated like in the examples shown below.

When you use it to extend functionality of an existing cover implementation, this component will generate a new cover entity with these new features.

The component adds two services ```set_known_position``` and ```set_known_action``` which allow updating HA in response to external stimuli like sensors. 

[Support forum](https://community.home-assistant.io/t/custom-component-cover-time-based/187654/3?u=robi)

## Installation
[![hacs_badge](https://img.shields.io/badge/HACS-Default-orange.svg?style=for-the-badge)](https://github.com/custom-components/hacs)
* Install using HACS, or manually: copy all files in custom_components/cover_rf_time_based to your <config directory>/custom_components/cover_rf_time_based/ directory.
* Restart Home-Assistant.
* Create the required scripts in scripts.yaml.
* Add the configuration to your configuration.yaml.
* Restart Home-Assistant again.

### Usage
To use this component in your installation, you have to set RF-sending scripts to open, close and stop the cover (see below), and add the following to your configuration.yaml file:

### Example configuration.yaml entry
```yaml
cover:
  - platform: cover_rf_time_based
    devices:
      my_room_cover_time_based:
        name: My Room Cover
        travelling_time_up: 36
        travelling_time_down: 34
        close_script_entity_id: script.rf_myroom_cover_down
        stop_script_entity_id: script.rf_myroom_cover_stop
        open_script_entity_id: script.rf_myroom_cover_up
        send_stop_at_ends: False #optional
        always_confident: False #optional
```
  
**OR**:  
  
```yaml
cover:
  - platform: cover_rf_time_based
    devices:
      my_room_cover_time_based:
        name: My Room Cover
        travelling_time_up: 36
        travelling_time_down: 34
        cover_entity_id: cover.myroom
        send_stop_at_ends: True #optional
        always_confident: True #optional
```
All mandatory settings self-explanatory. 

Optional settings:
- `send_stop_at_ends` defaults to `False`. If set to `True`, the Stop script will be run after the cover reaches to 0 or 100 (closes or opens completely). This is for people who use interlocked relays in the scripts to drive the covers, which need to be released when the covers reach the end positions.
- `always_confident` defaults to `False`. If set to `True`, the calculated state will always be assumed to be accurate. This mainly affects UI components - for example, if the cover is fully opened, the open button will be disabled. Make sure to [set](#cover_rf_time_basedset_known_position) the current state after first setup and only use this entity to control the covers. Not recommended to be `True` for RF-based covers.


### Example scripts.yaml entry
#### RF covers
The following example assumes that you're using an RF bridge running [Tasmota](https://tasmota.github.io/docs/devices/Sonoff-RF-Bridge-433/) or ESPHome open source firmware to integrate your radio-controlled covers. The command scripts pass the `rfraw_data` parameter to a general transmitter script which takes care of queuing the transmission of the codes and keeping an appropriate delay between them:
```yaml
'rf_transmitter':
  alias: 'RF Transmitter'
  mode: queued
  max: 30
  sequence:
    # With Tasmota
    - service: mqtt.publish
      data:
        topic: 'cmnd/rf-bridge-1/rfraw'
        payload: '{{ rfraw_data }}'
    # With ESPHome
    - service: esphome.sonoff_rf_bridge_send_rf_raw
      data:
        raw: '{{ rfraw_data }}'
    # add a little delay
    - delay: 00:00:01

'rf_myroom_cover_down':
  alias: 'RF send MyRoom Cover DOWN'
  mode: single
  max_exceeded: silent
  sequence:
    - service: script.turn_on
      target:
        entity_id: script.rf_transmitter
      data:
        variables:
          rfraw_data: 'AAB0XXXXX....XXXXXXXXXX'

'rf_myroom_cover_stop':
  alias: 'RF send MyRoom Cover STOP'
  mode: single
  max_exceeded: silent
  sequence:
    - service: script.turn_on
      target:
        entity_id: script.rf_transmitter
      data:
        variables:
          rfraw_data: 'AAB0XXXXX....XXXXXXXXXX'

 'rf_myroom_cover_up':
  alias: 'RF send MyRoom Cover UP'
  mode: single
  max_exceeded: silent
  sequence:
    - service: script.turn_on
      target:
        entity_id: script.rf_transmitter
      data:
        variables:
          rfraw_data: 'AAB0XXXXX....XXXXXXXXXX'
```

For the scripts above with Tasmota you need a small automation in **automations.yaml** to set `RfRaw` back to `0` to avoid spamming your MQTT server with loads of sniffed raw RF data. This trigger is checked every minute only so set `> 40` set in the `value_template` to be a bit bigger than your biggest `travelling_time`:

```yaml
- id: rf_transmitter_cancel_sniff
  alias: 'RF Transmitter cancel sniffing'
  trigger:
    platform: template
    value_template: "{{ ( as_timestamp(now()) - as_timestamp(state_attr('script.rf_transmitter', 'last_triggered')) | int(0) ) > 40 }}"
  action:
    - service: mqtt.publish
      data:
        topic: 'cmnd/rf-bridge-1/rfraw'
        payload: '0'
```
#### Switched covers
The example below assumes you've set `send_stop_at_ends: True` in the cover config, and you're using any [two-gang switch running Tasmota](https://tasmota.github.io/docs/devices/Sonoff-Dual-R2/) open source firmware to integrate your switch-controlled covers:
```yaml
'rf_myroom_cover_down':
  alias: 'Switches send MyRoom Cover DOWN'
  mode: single
  max_exceeded: silent
  sequence:
    - service: mqtt.publish
      data:
        topic: 'cmnd/myroomcoverswitch/POWER1' # open/close
        payload: 'OFF'
    - service: mqtt.publish
      data:
        topic: 'cmnd/myroomcoverswitch/POWER2' # power
        payload: 'ON'

'rf_myroom_cover_stop':
  alias: 'Switches send MyRoom Cover STOP'
  mode: single
  max_exceeded: silent
  sequence:
    - service: mqtt.publish
      data:
        topic: 'cmnd/myroomcoverswitch/POWER2' # power
        payload: 'OFF'
    - service: mqtt.publish
      data:
        topic: 'cmnd/myroomcoverswitch/POWER1' # open/close
        payload: 'OFF'

'rf_myroom_cover_up':
  alias: 'Switches send MyRoom Cover UP'
  mode: single
  max_exceeded: silent
  sequence:
    - service: mqtt.publish
      data:
        topic: 'cmnd/myroomcoverswitch/POWER1' # open/close
        payload: 'ON'
    - service: mqtt.publish
      data:
        topic: 'cmnd/myroomcoverswitch/POWER2' # power
        payload: 'ON'
```
Note how you don't have to configure these as switches in Home Assistant at all, it's enough just to publish MQTT commands strainght from the script (credits to [VDRainer](https://github.com/VDRainer) for this example).
Of course you can customize based on what ever other way to trigger these 3 type of movements. You could, for example, turn on and off warning lights along with the movement.

### Services to set position or action without triggering cover movement

This component provides 2 services:

1.  ```cover_rf_time_based.set_known_position``` lets you specify the position of the cover if you have other sources of information, i.e. sensors. It's useful as the cover may have changed position outside of HA's knowledge, and also to allow a confirmed position to make the arrow buttons display more appropriately.
1.  ```cover_rf_time_based.set_known_action``` is for instances when an action is caught in the real world but not process in HA, .e.g. an RF bridge detects a ```stop``` action that we want to input into HA without calling the stop command.


#### ```cover_rf_time_based.set_known_position```

In addition to ```entity_id``` and ```position``` takes 2 optional parameters:
* ```confident``` that affects how the cover is presented in HA. Setting confident to ```true``` will mean that certain button operations aren't permitted.
* ```position_type``` allows the setting of either the ```target``` or ```current``` posistion.

Following examples to help explain parameters and use cases:

1.  This example automation uses ```position_type: current``` and ```confident: true``` when a reed sensor has indicated a garage door is closed when contact is made:

```yaml
- id: 'garage_closed'
  alias: 'Doors: garage set closed when contact'
  trigger:
  - entity_id: binary_sensor.door_garage_cover
    platform: state
    to: 'off'
  condition: []
  action:
  - data:
      confident: true
      entity_id: cover.garage_door
      position: 0
      position_type: current
    service: cover_rf_time_based.set_known_position
``` 

We have set ```confident``` to ```true``` as the sensor has confirmed a final position. The down arrow is now no longer available in default  HA frontend when the cover is closed. 
```position_type``` of ```current``` means the current position is moved immediately to 0 and stops there (provided cover is not moving, otherwise will contiune moving to original target). 


2.  This example uses ```position_type: target``` (the default) and ```confident: false``` (also default) where an RF bridge has interecepted an RF command, so we know an external remote has triggered cover opening action:

```yaml
- id: 'rf_cover_opening'
  alias: 'RF_Cover: set opening when rf received'
  trigger:
  - entity_id: sensor.rf_command
    platform: state
    to: 'open'
  condition: 
  - condition: state
    entity_id: cover.rf_cover
    state: closed
  action:
  - data:
      entity_id: cover.rf_cover
      position: 100
    service: cover_rf_time_based.set_known_position
```

```confident``` is omitted so defaulted to ```false``` as we're not sure where the movement may end, so all arrows are available.
```position_type``` is omitted so defaulted to ```target```, meaning cover will transition to ```position``` without triggering any start or stop actions.

#### ```cover_rf_time_based.set_known_action```
This service mimics cover movement in Home Assistant without actually sending out commands to the cover. It can be used for example when external RF remote controllers act on the cover directly, but the signals can be captured with an RF brigde and Home Assistant can play the movement in parrallel with the real cover. In addtion to ```entity_id``` takes parameter ```action``` that should be one of open, close or stop.

Example:

```yaml
- id: 'rf_cover_stop'
  alias: 'RF_Cover: set stop action from bridge trigger'
  trigger:
  - entity_id: sensor.rf_command
    platform: state
    to: 'stop'
  condition:[]
  action:
  - data:
      entity_id: cover.rf_cover
      action: stop
    service: cover_rf_time_based.set_known_action
```
In this instance we have caught a stop signal from the RF bridge and want to update HA cover without triggering another stop action.

### Icon customization
For proper icon display (opened/moving/closed) customization can be added to `configuration.yaml` based of what type of covers you have, either one by one, or for all covers at once:

```yaml
homeassistant:
  customize_domain: #for all covers 
     cover:
      device_class: shutter
  customize: #for each cover separately
    cover.my_room_cover_time_based:
      device_class: shutter
```
More details in [Home Assistant device class docs](https://www.home-assistant.io/docs/configuration/customizing-devices/#device-class).

### Some tips 

#### When using this component with Tasmota RF Bridge in automations

Since there's no feedback from the cover about its current state, state is assumed based on the last command sent, and position is calculated based on the fraction of time spent travelling up or down. You need to measure time by opening/closing the cover using the original remote controller, not through the commands sent from Home Assistant (as they may introduce some delay).

Tasmota RF bridge is able to send out the radio-frequency commands very quickly. If some of your covers 'miss' the commands occassionally (you can see that from the fact that the state shown in Home Assistant does not correspond to reality), it may be that those cover motors do not understand the codes when they are sent 'at once' from Home Assistant. 

This can be handled in multiple ways:
- try increasing your RF range. Make sure the wire antennas of the covers are not tied close to the power cables or to big metallic surfaces. For 433MHz, the antenna length should be around 17cm (this may include the part going inside the tube motor). Sonoff RF Bridge has two copper helical antennas near the PCB, you can unsolder them and simply solder in place two straight hard wires of 17.3cm, which can go out through some small holes on the sides of the unit. You need to solder only one end of each wire, to the points where the helical legs were shorter (points U7 and U8). This will increase the range substantially, to the cost of aesthetics.
- avoid _backlogs_ with `rfraw AAB0XXXXX....XXXXXXXXXX; rfraw 0` if you need multiple covers opening and closing at once. Switching the sniff on and off quickly for every cover movement may cause issues. It's enough to send `rfraw 0` only once with some delay after all procedures related to cover movements finished, the example scripts above take care of that.
- if you are sending `0xB0` codes (decoded with [BitBucketConverter.py](https://github.com/Portisch/RF-Bridge-EFM8BB1)) you can tweak those to be sent with repetitions (multiple times) by changing the repetition parameter (5th byte) of the code. [For example](https://github.com/arendst/Tasmota/issues/5936#issuecomment-500236581) 20 repetitions can be achieved by changing 5th byte from 04 to 14. Also BitBucketConverter can be run by specifiying the required repetitions at command line before decoding. Some covers might not like this, though.
- alternatively, you can further reduce stress by making sure you don't use [cover groups](https://www.home-assistant.io/integrations/cover.group/) containing multiple covers provided by this integration, and also in automation don't include multipe covers separated by commas in one service call. You could create separate service calls for each cover, moreover, add more delay between them:
```yaml
- alias: 'Covers down when getting dark'
  mode: single
  max_exceeded: silent
  trigger:
    - platform: numeric_state
      below: 400
      for: "00:05:00"
      entity_id: sensor.outside_light_sensor
  action:
    - service: cover.close_cover
      entity_id: cover.room_1
    - delay: '00:{{ (range(1,10)|random|int) }}:00'
    - service: cover.close_cover
      entity_id: cover.room_2
    - delay: '00:00:02'
    - service: cover.set_cover_position
      data:
        entity_id: cover.room_3
        position: 20
    - delay: '00:00:01'
    - service: cover.set_cover_position
      data:
        entity_id: cover.room_4
        position: 30
```

#### Implementation with ESPHome
  
Use the following configuration for ESPHome on Sonoff RF Bridge:
  
```yaml
substitutions:
  device_name: sonoff-rf-bridge
  friendly_name: "Sonoff RF Bridge"
  device_ip: 192.168.81.22

esphome:
  name: ${device_name}
  platform: ESP8266
  board: esp01_1m

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  manual_ip:
    static_ip: ${device_ip}
    gateway: 192.168.81.254
    subnet: 255.255.255.0

logger:
  baud_rate: 0

uart:
  tx_pin: GPIO01
  rx_pin: GPIO03
  baud_rate: 19200

rf_bridge:

api:
  reboot_timeout: 15min
  password: !secret api_password
  encryption:
    key: !secret encryption_key
  services:
    - service: send_rf_raw
      variables:
        raw: string
      then:
        - rf_bridge.send_raw:
            raw: !lambda 'return raw;'

ota:
  password: !secret ota_password

web_server:
  port: 80
  auth:
    username: admin
    password: !secret web_server_password

sensor:
  - platform: wifi_signal
    name: ${friendly_name} WiFi signal
    update_interval: 60s
  - platform: uptime
    name: ${friendly_name} Uptime

status_led:
  pin:
    number: GPIO13
    inverted: true

binary_sensor:
- platform: status
  name: ${friendly_name} State

switch:
- platform: restart
  name: ${friendly_name} Restart
  
```
