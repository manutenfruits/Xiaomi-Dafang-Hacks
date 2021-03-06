## Xiaomi Dafang Integration in Home Assistant





### On the Home Assistant side

First let's set up your camera stream. Make sure the rtsp-h264 service in the [service control panel](http://dafang/cgi-bin/scripts.cgi) is running and you can connect to [it](rtsp://dafang:8554/unicast) via e.g. vlc player.

![service control panel](services_panel.png)

Then you can integrate the rtsp stream using the camera ffmpeg component by adding the following lines to your camera.yaml:

```yaml
- platform: ffmpeg
  name: DaFang3
  input: -rtsp_transport tcp -i rtsp://dafang:8554/unicast
```

Most other sensors & actors are easily integrated via [mqtt discovery](https://www.home-assistant.io/docs/mqtt/discovery/). If everything works, integration looks like this (grouped into one group):

![MQTT discovery  result](mqtt_autodiscovery.png)

To enable mqtt discovery in Home Assistant please add/adjust in your .homeassistant/configuration.yaml:
```yaml
mqtt:
  broker: localhost
  discovery: true
  discovery_prefix: homeassistant
```
and restart your Home Assistant instance. Apparently this does not work with Home Assistant's internal mqtt broker, so better [run your own](https://www.home-assistant.io/docs/mqtt/broker/#run-your-own).

### On the Xiaomi Dafang Camera side:

Connect to your camera via ssh (or your preferred ftp client):
```shell
ssh root@dafang # default password is ismart12
```

copy /system/sdcard/config/mqtt.dist to /system/sdcard/config/mqtt.conf:
```shell
cp /system/sdcard/config/mqtt.dist /system/sdcard/config/mqtt.conf
```
Set up your broker, LOCATION and DEVICE_NAME
and uncomment AUTODISCOVERY_PREFIX (only then the dafang configurations will be published):

```shell
vi /system/sdcard/config/mqtt.conf
```
Press `i` to enter insert mode. Once you are done hit `ESC` and enter `:wq` to write your changes.

Restart the mqtt-status & mqtt-control services in the [service control panel](http://dafang/cgi-bin/scripts.cgi) to make them pick up on your changes.

![service control panel](services_panel.png)

 In case your Home Assistant needs to be restarted, changes are not persisted in any configuration file and the mqtt discovery configuration has to be resent from the camera. This can be enforced by restarting the mqtt-control service.

To put all the sensors & actors conveniently into one group you can use the following template:

```yaml
Dafang3:
    - camera.dafang3
    - switch.dafang3_rtsp_server
    - sensor.dafang3
    - device_tracker.dafang3
    - sensor.dafang3_light_sensor
    - switch.dafang3_ir_filter
    - switch.dafang3_ir_led
    - switch.dafang3_night_mode
    - switch.dafang3_night_mode_auto
    - switch.dafang3_blue_led
    - switch.dafang3_yellow_led
    - switch.dafang3_motion_detection
    - switch.dafang3_motion_tracking
    - camera.dafang3_motion_snapshot
    - binary_sensor.dafang3_motion_sensor
```

### To set up mqtt motion detection alerts:

copy /system/sdcard/config/motion.dist to /system/sdcard/config/motion.conf:
```shell
cp /system/sdcard/config/motion.dist /system/sdcard/config/motion.conf
```

Set up the motion detection via its [webinterface](http://dafang/configmotion.html).

In motion.conf define how your camera should react on motion events:
```shell
vi /system/sdcard/config/motion.conf
```
For your camera to send mqtt motion detection messages it should be enabled by setting:
```
publish_mqtt_message=true
```

You should now be getting messages on topic `myhome/mycamera/motion` and images on `myhome/mycamera/motion/snapshot` while    `myhome/mycamera/motion/detection` is set to ON

To react on a motion event, in your automations.yaml define something like:

```yaml
- action:
  - data:
      message: detected
      title: motion
    service: notify.me
  alias: motion detector
  condition: []
  id: '9522507257594'
  trigger:
  - payload: 'ON'
    platform: mqtt
    topic: myhome/mycamera/motion
```
