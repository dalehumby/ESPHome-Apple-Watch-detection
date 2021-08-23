# ESPHome BLE Apple Watch presence detection

An [ESPHome configuration]() to reliably detect an Apple Watch for room level presence detection by Home Assistant.

## How BLE tracking works
To track a person or object, you attach a [BLE sticker](). Every few seconds the sticker broadcasts its presence. It is identified through its unique MAC address. A BLE receiver, like the Raspberry Pi running [room assistant](), or ESP32 with [ESPHome BLE RSSI sensor]() detects the broadcast as well as the received signal strength ([RSSI]()). If the signal is weak then the tag is far away, and if the signal is strong then the tag is near. You can use this to infer whether the tag is in the room or not, and base home automation on the tags presence or absence.

For personal tracking I didn't want to wear a tag. I do always wear my Apple Watch, and wanted to use that for presence detection. But to prevent tracking, Apple devices periodically change their MAC address, so you cannot rely on the MAC address alone for presence tracking. 

## Apple Watch BLE tracking
Apple devices use BLE broadcasts to notify other Apple devices of their proximity, for use in handoff, Find My and Airdrop. This is called the [Apple Continuity Protocol](https://github.com/furiousMAC/continuity) and has been somewhat reverse engineered.

Apple Watch broadcasts the [Nearby Info](https://github.com/furiousMAC/continuity/blob/master/messages/nearby_info.md) message every few seconds, which has enough information to reliably find it amongst all other BLE broadcast messages.

### Apple Watch Nearby Info message
Broadcast message (in hex) split for easier reading: `4c00 10 05 01 98 86b356`, where

- `4c00` is Apple's identifier. All Apple BLE broadcasts start with this
- `10` identifies the Nearby Info message
- `05` is the length: 5 bytes to follow
- `01` is the status flags and action code. Always `01`
- `98` the Data flags. If the watch is unlocked the is `98` and if locked (e.g. on night stand, not on wrist) then is `18`
- `86b356` is 3 byte Authentication tag. There is no useful info here as it changes continuously

I can reliably detect my Apple Watch by searching all broadcast messages for Apple manufacturer ID `4c00` with data fields that starts with `10 05 01 98`.

Although I have not yet done this, you can confirm that it is _your_ Apple watch by connecting to the device, listing all services, and then going through each service looking for the characteristic ID `Model Number String` that has a value such as `Watch3,3`. 

As long as you're the only person with a series 3 Apple Watch with hardware revision 3 in your house, you will reliably be able to track yourself.

You can then store the MAC address until it cycles in ~45 minutes. When the MAC does change you rescan for the magic string `4c00 10 05 98` and service characteristic `Model Number String` `Watch3,3`.

## Tracking Apple Watch with ESPHome
In each room where I want BLE tracking I have an [ESP32 D1 mini]() running ESPHome. I have included [`lounge.yaml`]() as an example. Each ESP32 continually scans for the magic BLE string, and when it finds it, gets the signal strength (RSSI). The strength fluctuates depending where you are in the room and how much of your body is shielding the watch from the receiver. A number of filters reduce the noise, and then make a decision whether you are in the room or not. The RSSI and presence is sent to Home Assistant via the native API, and also published to MQTT for use by e.g. Node-RED.

### Config file explanation
I have templatised the yaml as much as possible. 

```
substitutions:
  roomname: bedroom
  static_ip: 10.0.0.16
  yourname: Dale
  rssi_present: "-73"
  rssi_not_present: "-85"
```

`roomname`, `static_ip` and `yourname` should be self explanatory. 

`rssi_present` is the signal strength that determines whether you are definitely in the room. Anything stronger than `-73` and you are present. For `rssi_not_present`, an RSSI of below `-85` means that you are definitely not present in the room.

The detection works as follows:

```
esp32_ble_tracker:
  scan_parameters:
    interval: 1.2s
    window: 500ms
    active: false
  on_ble_advertise:
    - then:
      # Look for manufacturer data of form: 4c00 10 05 01 98 XXXXXX
      - lambda: |-
          for (auto data : x.get_manufacturer_datas()) {
            if (data.uuid.to_string() == "0x004C" && data.data[0] == 0x10 && data.data[1] == 0x05 && data.data[2] == 0x01 && data.data[3] == 0x98) {
              int16_t rssi = x.get_rssi();
              id(apple_watch_rssi).publish_state(rssi);
              ESP_LOGD("ble_adv", "Found Apple Watch, rssi %i", rssi);
            }
          }
```

This uses ESPHome's [`esp32_ble_tracker`]() sensor. I've increased the `interval` and `window` to give as much time to detect the watch broadcast, but also enough time for the ESP32 to switch to WiFi to send results to MQTT and HA. We only listen for broadcasts, so `active: false`.

On all broadcasts, a lambda is run which looks for the manufacturer UUID `004c` (note: big-endian), and manufacturer data that starts with `10 05 01 98`. I am specifically looking for when the watch is **unlocked** (i.e. on my wrist) so that tracking stops when I take my watch off and it auto-locks. Locked state changes byte 3 to `18`. (All values hex.)

The lambda publishes the RSSI to the `apple_watch_rssi` template sensor:

```
  - platform: template
    id: apple_watch_rssi
    name: "$yourname Apple Watch $roomname RSSI"
    device_class: signal_strength
    unit_of_measurement: dBm
    accuracy_decimals: 0
    filters:
      - exponential_moving_average:
          alpha: 0.3
          send_every: 1
    on_value:
      then:
        - lambda: |-
            if (id(apple_watch_rssi).state > $rssi_present) {
              id(room_presence_debounce).publish_state(1);
            } else if (id(apple_watch_rssi).state < $rssi_not_present) {
              id(room_presence_debounce).publish_state(0);
            }
        - script.execute: presence_timeout  # Publish 0 if no rssi received
```

This applies an [`exponential_moving_average`]() filter. The `alpha` is fairly high to make it responsive when I actually change rooms, but just 'forgetful' enough to smooth out a lot of the random RSSI fluctuations.

Next apply hysteresis, so that only if the signal is stronger than `rssi_present` are you present (publish `1` to `room_presence_debounce` sensor), and only not present (publish `0`) when signal strength of the watch is below `rssi_not_present`. If the RSSI is between these two values then it does not publish anything, and the system will stay its current state: You either stay present or not present.

We also start a `presence_timeout` script, so that if no signal is detected (maybe you leave the house), then you publish a `0` value to indicate not present. 

```
script:
  # Publish event every 30 seconds when no rssi received
  id: presence_timeout
  mode: restart
  then:
    - delay: 30s
    - lambda: |-
        id(room_presence_debounce).publish_state(0);
    - script.execute: presence_timeout
```

The script is called each time an RSSI value is published. A 30 second delay timer is started. If the script is called again before the 30s delay is over, because of `mode: restart` the script is restarted, and the delay time is reset to 30s and we never publishing a `0` state. If no RSSI value has been received for 30 s then we publish a state of not present (`0`), and call the script again. Not present will continue to be published every 30s for as long as the Apple Watch is not detected.

Next is the debounce sensor:

```
  - platform: template
    id: room_presence_debounce
    filters:
      - sliding_window_moving_average:
          window_size: 3
          send_every: 1
```

in conjunction with the room presence binary sensor:

```
binary_sensor:
  - platform: template
    id: room_presence
    name: "$yourname $roomname presence"
    device_class: occupancy
    lambda: |-
      if (id(room_presence_debounce).state > 0.99) {
        return true;
      } else if (id(room_presence_debounce).state < 0.01) {
        return false;
      } else {
        return id(room_presence).state;
      }
```

The debounce sensor requires 3 consecutive readings to be the same to change presence. I use `0.99` and `0.01` in case there are floating point issues with the `[sliding_window_moving_average]()` filter.


## Integration to Home Assistant

### Automations

### Device Tracker


## Tested with
- ESPHome 2021.8
- Generic MINI32 board (ESP32 with integrated 5V to 3V and USB to serial converter)
- Apple Watch Series 3 (`Watch3,3`)

## Prior work on fingerprinting Apple device
- [Apple device continuity](https://github.com/furiousMAC/continuity)
- https://petsymposium.org/2020/files/papers/issue1/popets-2020-0003.pdf
- https://arxiv.org/pdf/1703.02874.pdf
- https://samteplov.com/uploads/shmoocon20/slides.pdf
- https://i.blackhat.com/eu-19/Thursday/eu-19-Yen-Trust-In-Apples-Secret-Garden-Exploring-Reversing-Apples-Continuity-Protocol-3.pdf

## Similar tracking projects
- [room-assistant](https://www.room-assistant.io/guide/) using Raspberry Pi Zero
- [ESP32-mqtt-room](https://github.com/jptrsn/ESP32-mqtt-room)

## Contributing
I've only tested this on my Series 3 Apple Watch, and not sure whether this works on other Apple Watches. Probably the easiest way to see BLE broadcasts is by running `$ sudo blescan -t 70` on your computer, rather than compiling ESPHome each time.

I'd prefer that this was a custom ESPHome sensor, similar to [Xiaomi Mijia BLE Sensors](https://esphome.io/components/sensor/xiaomi_ble.html), instead of 100 lines of yaml. If you can help with the C, please let me know.

All feedback welcome.
