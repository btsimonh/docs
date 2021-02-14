Bluetooth Low Energy in Tasmota consists of:

## ESP8266 or ESP32 via HM-1x or nRF24L01(+)
This allows for the receiving of BLE advertisments from BLE devices, including "iBeacons"
For this style, 

`#undef USE_BLE_ESP32`

## ESP32 native Bluetooth Low Energy support
This allows for the receiving of BLE advertisments from BLE devices, including "iBeacons" and BLE sensors, but also for the control of simple BLE devices, providing for reading, writing and receiving notifications. 
For this style, 

`#define USE_BLE_ESP32` 

Be aware, enabling of the native BLE on ESP32 has an impact on wifi performance.  Although later SDK helped a bit, expect more lag on the web interface and on MQTT.
If only controlling BLE devices, then scanning can be disabled, which will minimise wifi impact.

This is compiled by default in the Sensors firmware, but you still need to enable it using the web interface `configure BLE` button or setoption115 1.


!!! info "Presence detection with iBeacons or BLE sensor gateway using HM-1x or nRF24L01(+) peripherals"

## iBeacon  

!!! info "This feature is included only in tasmota-sensors.bin"
Otherwise you must [compile your build](Compile-your-build). Add the following to `user_config_override.h`:

### For ESP8266 or ESP32 via HM-1x or nRF24L01(+)
```
#undef USE_BLE_ESP32
#define USE_IBEACON          // Add support for bluetooth LE passive scan of ibeacon devices 
```
----
  
Tasmota uses a BLE 4.x module to scan for [iBeacon](https://en.wikipedia.org/wiki/IBeacon) devices. This driver is working with [HM-10 and clones](HM-10) and [HM16/HM17](HM-17) Bluetooth modules and potentially with other HM-1x modules depending on firmware capabilities.

!!! tip
    If using an extenral module, When first connected some modules will be in peripheral mode. You have to change it to central mode using commands `Sensor52 1` and `Sensor52 2`.

### For ESP32 built-in Bluetooth

You must [compile your build](Compile-your-build) for the ESP32 (since v9.1). Change the following to `user_config_override.h`:

```
#ifdef ESP32
  #define USE_BLE_ESP32
  #define USE_IBEACON_ESP32    // Use internal ESP32 Bluetooth module
#endif // ESP32
```
(note that when using `USE_BLE_ESP32`, you can combine iBEacon features with other BLE features)

### Features
For a list of all available commands see [Sensor52](Commands.md#sensor52) command.  

This driver reports all beacons found during a scan with its ID (derived from beacon's MAC address) prefixed with `IBEACON_` and RSSI value.

Every beacon report is published as an MQTT tele/%topic%/SENSOR in a separate message:

```json
tele/ibeacon/SENSOR = {"Time":"2021-01-02T12:08:40","IBEACON":{"MAC":"A4C1387FC1E1","RSSI":-56,"STATE":"ON"}}
```

If the beacon can no longer be found during a scan and the timeout interval has passed the beacon's RSSI is set to zero (0) and it is no longer displayed in the webUI

```json
tele/ibeacon/SENSOR = {"Time":"2021-01-02T12:08:40","IBEACON":{"MAC":"A4C1387FC1E1","RSSI":-56,"STATE":"OFF"}}
```

Additional fields will be present depending upon the beacon, e.g. NAME, UID, MAJOR, MINOR.


### Supported Devices
<img src="../_media/bluetooth/nRF51822.png" width=155 align="right">

All Apple compatible iBeacon devices should be discoverable. 

Various nRF51822 beacons should be fully Apple compatible, programmable and their battery lasts about a year.

- [Amazon.com](https://www.amazon.com/s?k=nRF51822+4.0)
- [Aliexpress](https://www.aliexpress.com/af/NRF51822-beacon.html)


Cheap "iTag" beacons with a beeper. The battery on these lasts only about a month.

- [Aliexpress](https://www.aliexpress.com/af/itag.html?trafficChannel=af&SearchText=itag&ltype=affiliate&SortType=default&g=y&CatId=0)
- [eBay](https://www.ebay.de/sch/i.html?_from=R40&_trksid=m570.l1313&_nkw=Smart-Tag-GPS-Tracker-Bluetooth-Anti-verlorene-Alarm-Key-Finder-Haustier-Kind&_sacat=0)
- [Amazon.com](https://www.amazon.com/s?k=itag+tracker+4.0)

<img src="../_media/bluetooth/itag.png" width=225><img src="../_media/bluetooth/itag2.png" width=225><img src="../_media/bluetooth/itag3.png" width=225>

!!! tip
    You can activate a beacon with a beeper using command `IBEACON_%BEACONID%_RSSI 99` (ID is visible in webUI and SENSOR reports). This command can freeze the Bluetooth module and beacon scanning will stop. After a reboot of Tasmota the beacon will start beeping and scanning will resume. (untested on ESP32 native BLE)
  
  
  
## Bluetooth Low Energy Sensors

Different vendors offer Bluetooth solutions as part of the XIAOMI family often under the MIJIA-brand (while AQUARA is the typical name for a ZigBee sensor).  
The sensors supported by Tasmota use BLE (Bluetooth Low Energy) to transmit the sensor data, but they differ in their accessibilities quite substantially.  
  
Basically all of them use of so-called „MiBeacons“ which are BLE advertisement packets with a certain data structure, which are broadcasted by the devices automatically while the device is not in an active bluetooth connection.  
The frequency of these messages is set by the vendor and ranges from one per 3 seconds to one per hour (for the battery status of the LYWSD03MMC). Motion sensors and BLE remote controls start to send when an event is triggered.  
These packets already contain the sensor data and can be passively received by other devices and will be published regardless if a user decides to read out the sensors via connections or not. Thus the battery life of a BLE sensor is not influenced by reading these advertisements and the big advantage is the power efficiency as no active bi-directional connection has to be established. The other advantage is, that scanning for BLE advertisements can happen nearly parallel (= very quick one after the other), while a direct connection must be established for at least a few seconds and will then block both involved devices for that time.  
This is therefore the preferred option, if technically possible (= for the supported sensors).
  
Most of the „older“ BLE-sensor-devices use unencrypted messages, which can be read by all kinds of BLE-devices or even a NRF24L01. With the arrival of "newer" sensors came the problem of encrypted data in MiBeacons, which can be decrypted in Tasmota (not yet with the HM-1x).  
Meanwhile it is possible to get the needed "bind_key" with the help of an open-source project: https://atc1441.github.io/TelinkFlasher.html  
At least the LYWSD03 allows the use of a simple BLE connection without any encrypted authentication and the reading of the sensor data using normal subscription methods to GATT-services (currently used on the HM-1x). This is more power hungry than the passive reading of BLE advertisements.  
Other sensors like the MJYD2S are not usable without the "bind_key".  
  
### Supported Devices

!!! note "It can not be ruled out, that changes in the device firmware may break the functionality of this driver completely!"  

The naming conventions in the product range of bluetooth sensors in XIAOMI-universe can be a bit confusing. The exact same sensor can be advertised under slightly different names depending on the seller (Mijia, Xiaomi, Cleargrass, ...).

 <table>
  <tr>
    <th class="th-lboi">MJ_HT_V1</th>
    <th class="th-lboi">LYWSD02</th>
    <th class="th-lboi">CGG1</th>
    <th class="th-lboi">CGD1</th>
  </tr>
  <tr>
    <td class="tg-lboi"><img src="../_media/bluetooth/mj_ht_v1.png" width=200></td>
    <td class="tg-lboi"><img src="../_media/bluetooth/LYWDS02.png" width=200></td>
    <td class="tg-lboi"><img src="../_media/bluetooth/CGG1.png" width=200></td>
    <td class="tg-lboi"><img src="../_media/bluetooth/CGD1.png" width=200></td>
  </tr>
  <tr>
    <td class="tg-lboi">temperature, humidity, battery</td>
    <td class="tg-lboi">temperature, humidity, battery</td>
    <td class="tg-lboi">temperature, humidity, battery</td>
    <td class="tg-lboi">temperature, humidity, battery</td>
  </tr>
    <tr>
    <td class="tg-lboi">passive for all entities, reliable battery value</td>
    <td class="tg-lboi">battery only active, thus not on the NRF24L01, set clock and unit, very frequent data sending</td>
    <td class="tg-lboi">passive for all entities, reliable battery value</td>
    <td class="tg-lboi">battery only active, thus not on the NRF24L01, no reliable battery value, no clock functions</td>
  </tr>
</table>  
  
 <table>
  <tr>
    <th class="th-lboi">MiFlora</th>
    <th class="th-lboi">LYWSD03MMC / ATC</th>
    <th class="th-lboi">NLIGHT</th>
    <th class="th-lboi">MJYD2S</th>
  </tr>
  <tr>
    <td class="tg-lboi"><img src="../_media/bluetooth/miflora.png" width=200></td>
    <td class="tg-lboi"><img src="../_media/bluetooth/LYWSD03MMC.png" width=200></td>
    <td class="tg-lboi"><img src="../_media/bluetooth/nlight.png" width=200></td>
    <td class="tg-lboi"><img src="../_media/bluetooth/mjyd2s.png" width=200></td>
  </tr>
  <tr>
    <td class="tg-lboi">temperature, illuminance, soil humidity, soil fertility, battery, firmware version</td>
    <td class="tg-lboi">temperature, humidity, battery</td>
    <td class="tg-lboi">motion</td>
    <td class="tg-lboi">motion, illuminance, battery, no-motion-time</td>
  </tr>
  <tr>
    <td class="tg-lboi">passive only with newer firmware (>3.0?), battery only active, thus not on the NRF24L01</td>
    <td class="tg-lboi">passive only with decryption or using custom ATC-firmware, no reliable battery value with stock firmware</td>
    <td class="tg-lboi">NRF24L01, ESP32</td>
    <td class="tg-lboi">passive only with decryption, thus only NRF24L01, ESP32</td>
  </tr>
</table>  
  
 <table>
  <tr>
    <th class="th-lboi">YEE RC</th>
    <th class="th-lboi">MHO-C401</th>
    <th class="th-lboi">MHO-C303</th>
  </tr>
  <tr>
    <td class="tg-lboi"><img src="../_media/bluetooth/yeerc.png" width=200></td>
    <td class="tg-lboi"><img src="../_media/bluetooth/MHO-C401.png" width=200></td>
    <td class="tg-lboi"><img src="../_media/bluetooth/MHO-C303.png" width=200></td>
  </tr>
  <tr>
    <td class="tg-lboi">button press (single and long)</td>
    <td class="tg-lboi">temperature, humidity, battery</td>
    <td class="tg-lboi">temperature, humidity, battery</td>
  </tr>
     <tr>
    <td class="tg-lboi">passive</td>
    <td class="tg-lboi">equal to the LYWS03MMC, but no custom firmware yet</td>
    <td class="tg-lboi">passive for all entities,  set clock and unit, no alarm functions, very frequent data sending</td>
  </tr>
</table> 
passive: data is received via BLE advertisments  
active: data is received via bidrectional connection to the sensor  
  
#### Devices with payload encryption  
  
The LYWSD03MMC, MHO-C401 and the MJYD2S will start to send advertisements with encrypted sensor data after pairing it with the official Xiaomi app (using TelinkFlasher to get the key also acts as a trigger to start sending?). Out-of-the-box the sensors do only publish a static advertisement.  
It is possible to do a pairing and get the necessary decryption key ("bind_key") here: https://atc1441.github.io/TelinkFlasher.html - note you do not have to flash the ATC firmware! 
This project also provides a custom firmware for the LYWSD03MMC, which then becomes an ATC and is supported by Tasmota too. Default ATC-setting will drain the battery more than stock firmware, because of very frequent data sending.  

For NRF based BLE:
This key and the corresponding MAC of the sensor can be injected with the NRFKEY-command (or NRFMJYD2S). It is probably a good idea to save the whole config as RULE like that:  
  
```haskell
rule1 on System#Boot do backlog NRFkey 00112233445566778899AABBCCDDEEFF112233445566; NRFkey 00112233445566778899AABBCCDDEEFF112233445566; NRFPage 6; NRFUse 0; NRFUse 4 endon
```  
(key for two sensors, 6 sensors per page in the WebUI, turn off all sensors, turn on LYWS03)  


For `USE_BLE_ESP32`, an encrypted sensor will show a link to the telelink flasher page, marked as 'NoKey' if an encrypted packet has been received and no key is present.

For the native BLE version of the driver (MI32), the key command is

`MI32Keys mac|alias=key mac|alias=key`
or
`MI32Key keymac`

where mac is a mac address, an alias may be used instead (see BLE commands).  The Key is the 32 character (16 byte) key retrieved by TelelinkFlasher.  `MI32Key is retains for b ackward compatibility, needing a 44 character combination of key and MAC.

LYWSD03MMC sends encrypted sensor data every 10 minutes. As there are no confirmed reports about correct battery presentation of the sensor (always shows 99%), this function is currently not supported.  
MJYD2S sends motion detection events and 2 discrete illuminance levels (1 lux or 100 lux for a dark or bright environment). Additionally battery level and contiguous time without motion in discrete growing steps (no motion time = NMT).    
 
  
#### Working principle of Tasmota BLE drivers (>8.5.)
  
The idea is to provide drivers with as many automatic functions as possible. Besides the hardware setup, there are zero or very few things to configure.  
The sensor namings are based on the original sensor names and shortened if appropriate (Flower care -> Flora). A part of the MAC will be added to the name as a suffix.  
All sensors are treated as if they are physically connected to the ESP8266 device. For motion and remote control sensors MQTT-messages will be published in (nearly) real time.
The ESP32 and the HM-1x-modules are real BLE devices whereas the NRF24L01 (+) is only a generic 2.4 GHz transceiver with very limited capabilities.  

With the `USE_BLE_ESP32` define set, the ESP32 Bluetooth allows for both iBeacon and Sensors to be used simultaneously in one build.
It also allows for some basic BLE configuration and veiwing through the web interface.


##### Options to read out the LYWSD03MMC  
  
1. Generate a bind_key  
The web-tool https://atc1441.github.io/TelinkFlasher.html allows the generation of a so-called bind_key by faking a pairing with the Xiaomi cloud. You can copy-paste this key and add the MAC to use this resultig key-MAC-string with key-command (NRFkey or MI32key). Then the driver will receive the sensors data roughly every 10 minutes (in two chunks for humidity and temperature with about a minute in between) and decode the data. This is the most energy efficient way. 
The current way of storing these keys on the ESP32 is to use RULES like that (for the NRF24L01 you would use NRFkey):  
```haskell
rule1 on System#Boot do backlog MI32key 00112233445566778899AABBCCDDEEFF112233445566; MI32key 00112233445566778899AABBCCDDEEFFAABBCCDDEEFF endon
```  
This option is currently not available for the HM-10 because of memory considerations as part of the standard sensor-firmware package.  

Note: in the latest Tasmota, a blue link will show next to a device if the device needs a key.  Clicking this likn will take you to a page guiding you to create a key, and at the end, puts that key into Tasmota, leaving you on a page with the required command to add to your rule. 

2. Flash custom ATC-firmware  
Use the same https://atc1441.github.io/TelinkFlasher.html to flash a custom ATC-firmware on the LYWSD03MMC. This will work out of the box with all three Tasmota-drivers. There is a slight chance of bricking the sensor, which would require some soldering and compiling skills to un-brick. This firmware does send data more frequently and is a little bit more power hungry than the stock firmware.  
There is also another new custom firmware here https://github.com/pvvx/ATC_MiThermometer with it's own flasher/config page.
The Custom mode is supported in latest Tasmota ESP32, but beware not to use 'All' mode.
  
3. Use active connections  
By default on the HM-10 (for legacy reasons) and at compile-time selectable on the ESP32 is the method to connect to the sensor from time to time. This circumvents the data encryption. This is very power hungry and drains the battery fast. Thus it is only recommended as fallback mechanism.

  
## BLE Sensors using HM-1x

!!! info "This feature is included only in tasmota-sensors.bin"
Otherwise you must [compile your build](Compile-your-build). Add the following to `user_config_override.h`:

```
#ifndef USE_HM10
#define USE_HM10          // Add support for HM-10 as a BLE-bridge (+9k3 code)
#endif
```

### Features
Supported sensors will be connected to at a set interval (default interval equals TelePeriod). A subscription is established for 5 seconds and data (e.g. temperature, humidity and battery) is read and reported to an mqtt topic (Dew point is calculated):

```json
tele/%topic%/SENSOR = {"Time":"2020-03-24T12:47:51","LYWSD03-52680f":{"Temperature":21.1,"Humidity":58.0,"DewPoint":12.5,"Battery":100},"LYWSD02-a2fd09":{"Temperature":21.4,"Humidity":57.0,"DewPoint":12.5,"Battery":2},"MJ_HT_V1-d8799d":{"Temperature":21.4,"Humidity":54.6,"DewPoint":11.9},"TempUnit":"C"}
```

After a completed discovery scan, the driver will report the number of found sensors. As Tasmota can not know how many sensors are meant to be discovered you have to force a re-scan until the desired number of devices is found.
```haskell
Rule1 ON HM10#Found<6 DO Add1 1 ENDON ON Var1#State<=3 DO HM10Scan ENDON 
```
This will re-scan up to 3 times if less than 6 sensors are found.

#### Commands

Command|Parameters
:---|:---
HM10Scan<a id="hm10scan"></a>|Start a new device discovery scan
HM10Period<a id="hm10period"></a>|Show interval in seconds between sensor read cycles. Set to TelePeriod value at boot.<BR>|`<value>` = set interval in seconds
HM10Baud<a id="hm10baud"></a>|Show ESP8266 serial interface baudrate (***Not HM-10 baudrate***)<BR>`<value>` = set baudrate
HM10AT<a id="hm10at"></a>|`<command>` = send AT commands to HM-10. See [list](http://www.martyncurrey.com/hm-10-bluetooth-4ble-modules/#HM-10%20-%20AT%20commands)
HM10Time <a id="hm10time"></a>|`<n>` = set time time of a **LYWSD02 only** sensor to Tasmota UTC time and timezone. `<n>` is the sensor number in order of discovery starting with 0 (topmost sensor in the webUI list).
HM10Auto <a id="hm10auto"></a>|`<value>` = start an automatic discovery scan with an interval of  `<value>` seconds to receive data in BLE advertisements periodically.<BR>This is a passive scan and does not produce a scan response from the BLE sensor. It does not increase the sensors battery drain.
HM10Page<a id="hm10page"></a>|Show the maximum number of sensors shown per page in the webUI list.<BR>`<value>` = set number of sensors _(default = 4)_
HM10Beaconx <a id="HM10beacon"></a>| Set a BLE device as a beacon using the (fixed) MAC-address<BR>x - set beacon 1 .. 4 <BR> x= 0 - will start a BLE scan and print result to the console <BR>`<value>` (12 or 17 characters) = use beacon given the MAC interpreted as a string `AABBCCDDEEFF` (also valid: `aa:BB:cc:dd:EE:FF`)  MAC of `00:00:00:00:00:00` will stop beacon x
  
  
## BLE Sensors using nRF24L01(+)

### Configuration
  
You must [compile your build](Compile-your-build). Change the following in `my_user_config.h`:

```
// -- SPI sensors ---------------------------------
#define USE_SPI                                  // Hardware SPI using GPIO12(MISO), GPIO13(MOSI) and GPIO14(CLK) in addition to two user selectable GPIOs(CS and DC)
#ifdef USE_SPI
  #define USE_NRF24                              // Add SPI support for NRF24L01(+) (+2k6 code)
  #ifdef USE_NRF24
    #define USE_MIBLE                            // BLE-bridge for some Mijia-BLE-sensors (+4k7 code)
```    
    
Sensors will be discriminated by using the Product-ID of the MiBeacon. A human readable short product name will be shown instead of the company-assigned ID of the BLE Public Device Address (= the "lower" 24 bits). 

A TELE message could like look this:  
  
```
10:13:38 RSL: stat/tasmota/STATUS8 = {"StatusSNS":{"Time":"2019-12-18T10:13:38","Flora-6ab577":{"Temperature":21.7,"Illuminance":21,"Humidity":0,"Fertility":0},"MJ_HT_V1-3108be":{"Temperature":22.3,"Humidity":56.1},"TempUnit":"C"}}
```
  
As the NRF24L01 can only read BLE-advertisements, only the data in these advertisements is accessible.  
All sensors have an additional GATT-interface with more data in it, but it can not be read with a NRF24l01. 
  
As we can not use a checksum to test data integrity of the packet, only data of sensors whose adresses showed up more than once (default = 3 times) will be published. 
Internally from time to time "fake" sensors will be created, when there was data corruption in the address bytes.  These will be removed automatically.  
  
  
#### Commands

Command|Parameters
:---|:---
NRFPage<a id="nrfpage"></a>|Show the maximum number of sensors shown per page in the webUI list.<BR>`<value>` = set number of sensors _(default = 4)_
NRFIgnore<a id="nrfignore"></a>|`0` = all known sensor types active_(default)_<BR>`<value>` =  ignore certain sensor type (`1` = Flora, `2` = MJ_HT_V1, `3` = LYWSD02, `4` = LYWSD03, `5` = CGG1, `6` = CGD1, `7` = NLIGHT,`8` = MJYD2S,`9` = YEERC (DEPRECATED, please switch to NRFUSE)
NRFUse<a id="nrfuse"></a>|`0` = all known sensor types inactive<BR>`<value>` =  ignore certain sensor type (`1` = Flora, `2` = MJ_HT_V1, `3` = LYWSD02, `4` = LYWSD03, `5` = CGG1, `6` = CGD1, `7` = NLIGHT,`8` = MJYD2S,`9` = YEERC
NRFScan<a id="nrfscan"></a>| Scan for regular BLE-advertisements and show a list in the console<BR>`0` = start a new scan list<BR>`1` = append to the scan list<BR>`2` = stop running scan
NRFBeacon<a id="nrfbeacon"></a>| Set a BLE device as a beacon using the (fixed) MAC-address<BR>`<value>` (1-3 digits) = use beacon from scan list<BR>`<value>` (12 characters) = use beacon given the MAC interpreted as an uppercase string `AABBCCDDEEFF`
NRFKey<a id="nrfkey"></a>| Set a "bind_key" for a MAC-address to decrypt (LYWSD03MMC & MHO-C401). The argument is a 44 uppercase characters long string, which is the concatenation of the bind_key and the corresponding MAC.<BR>`<00112233445566778899AABBCCDDEEFF>` (32 characters) = bind_key<BR>`<112233445566>` (12 characters) = MAC of the sensor<BR>`<00112233445566778899AABBCCDDEEFF112233445566>` (44 characters)= final string
NRFMjyd2s<a id="nrfmjyd2s"></a>| Set a "bind_key" for a MAC-address to decrypt sensor data of the MJYD2S. The argument is a 44 characters long string, which is the concatenation of the bind_key and the corresponding MAC.<BR>`<00112233445566778899AABBCCDDEEFF>` (32 characters) = bind_key<BR>`<112233445566>` (12 characters) = MAC of the sensor<BR>`<00112233445566778899AABBCCDDEEFF112233445566>` (44 characters)= final string
NRFNlight<a id="nrfnlight"></a>| Set the MAC of an NLIGHT<BR>`<value>` (12 characters) =  MAC interpreted as an uppercase string `AABBCCDDEEFF`
  
  
### Beacon  
  
A simplified presence dection will scan for regular BLE advertisements of a given BT-device defined by its MAC-address. It is important to know, that many new devices (nearly every Apple-device) will change its MAC every few minutes to prevent tracking.  
If the driver receives a packet from the "beacon" a counter will be (re-)started with an increment every second. This timer is published in the TELE-message, presented in the webUI and processed as a RULE.
The stability of regular readings will be strongly influenced by the local environment (many BLE-devices nearby or general noise in the 2.4-GHz-band). 

## BLE Sensors on ESP32 using built-in Bluetooth

Since Tasmota v9.1.0.1 this driver is part of the standard build 'tasmota32'. It must be enabled at runtime via `setoption115 1`. 
  
To turn on/off support for decyrption, change the following in the driver code:  

```haskell
#define USE_MI_DECRYPTION
```  
Without encryption support the driver will read data from found LYWSD03MMC via connection. This will increase power consumption of the sensor, but no "bind_key" is needed.  

The driver will start to scan for known sensors automatically using a hybrid approach, when encryption support is deactivated. In the first place MiBeacons are passively received and only found LYWSD03MMC- and MHO-C401-sensors will be connected at the given period to read data in order to be as energy efficient as possible.
Battery data is in general of questionable value for the LYWSD0x, CGD1, MHO-C401 and (maybe) Flora (some are even hard coded on the device to 99%). That's why only MJ_HT_V1, CGG1 (untested) and LYWSD03 (calculated battery level) will automatically update battery data. The other battery levels can be read by command. 
The internally very similiar LYWS03MMC and MHO-C401 behave very confusing in delivering battery status. Both report (fixed) 99% or 100% via encrypted payload or connected reading of the battery characteristic. But they can deliver a correct battery voltage in their payload of the temperature and humidity data, which will be mapped to a percentage level by the driver.  
  
#### Commands

Command|Parameters
:---|:---
MI32Period<a id="mi32period"></a>|Show interval in seconds between sensor read cycles for the LYWSD03. Set to TelePeriod value at boot.<BR>|`<value>` = set interval in seconds
MI32Time <a id="mi32time"></a>|`<n>` = set time time of a **LYWSD02 only** sensor to Tasmota UTC time and timezone. `<n>` is the sensor number in order of discovery starting with 0 (topmost sensor in the webUI list).
MI32Unit <a id="mi32unit"></a>|`<n>` = toggle the displayed temperature units of a **LYWSD02 only** sensor. `<n>` is the sensor number in order of discovery starting with 0 (topmost sensor in the webUI list).  Reporting of the temperature is always in Celcius, this only changes the value shown on the device.
MI32Page<a id="mi32page"></a>|Show the maximum number of sensors shown per page in the webUI list.<BR>`<value>` = set number of sensors _(default = 4)_
MI32Battery<a id="mi32battery"></a>|Reads missing battery data for LYWSD02, Flora and CGD1.
MI32Key<a id="mi32key"></a>| Set a "bind_key" for a MAC-address to decrypt sensor data (LYWSD03MMC, MJYD2S). The argument is a 44 uppercase characters long string, which is the concatenation of the bind_key and the corresponding MAC.<BR>`<00112233445566778899AABBCCDDEEFF>` (32 characters) = bind_key<BR>`<112233445566>` (12 characters) = MAC of the sensor<BR>`<00112233445566778899AABBCCDDEEFF112233445566>` (44 characters)= final string
MI32Beaconx <a id="mi32beacon"></a>| Set a BLE device as a beacon using the (fixed) MAC-address<BR>x - set beacon 1 .. 4 <BR> x=0 - will start a BLE scan and send result via TELE-message <BR>`<value>` (12 or 17 characters) = use beacon given the MAC interpreted as a string `AABBCCDDEEFF` (also valid: `aa:BB:cc:dd:EE:FF`)  MAC of `00:00:00:00:00:00` will stop beacon x
MI32Blockx <a id="mi32block"></a>| Ignore Xiaomi sensors using the (fixed) MAC-address<BR>x=1 - show block list<BR>x=0 - delete block list<BR> x=1 + MAC-address - add MAC to to be blocked to the block list<BR>x=0 + MAC-address - remove MAC to to be blocked to the block list<BR>`<value>` (12 or 17 characters) = MAC interpreted as a string `AABBCCDDEEFF` (also valid: `aa:BB:cc:dd:EE:FF`)
MI32Optionx 0/1<a id="mi32option"></a>| Set driver options at runtime<BR> x=0 - 0 -> sends only recently received sensor data, 1 -> aggregates all recent sensors data types<BR>x=1 - 0 -> shows full sensor data at TELEPERIOD, 1 -> shows no sensor data at TELEPERIOD<BR>x=2 - 0 -> sensor data only at TELEPERIOD (default and "usual" Tasmota style), 1 -> direct bridging of BLE-data to mqtt-messages
  
!!! tip 
If you really want to read battery for LYWSD02, Flora and CGD1, consider doing it once a day with a RULE:
`RULE1 on Time#Minute=30 do MI32Battery endon`
This will update every day at 00:30 AM.  
  
   
### Beacon  

(now removed if using BLE_ESP32)

A count-up-timer starts for every beacon a  with every received advertisement, starting with 0.  
  
TELE-output:  
`"Beacon1":{"MAC":"11:22:33:44:55:66","CID":"0x0000","SVC":"0x0000","UUID":"0x0000","Time":4,"RSSI":0}}`  
  
RULE-example:  
`on Beacon2#time==30 do SOMETHING endon` - is triggered 30 seconds after last packet was received  
`on system#boot do MI32Beacon2 AABBCCDDEEFF endon` - save configuration for beacon 2 
  
  
(following AD type, read here: https://www.bluetooth.com/specifications/assigned-numbers/generic-access-profile/)  
CID - company identifier  
SVC - service data  
UUID - service or class UUID  


### Notes on BLE Aliases

If using BLE_ESP32, then you may set Aliases for mac addresses using
`BLEAlias mac=name mac2=name2`
And then in most commands, you may refer to the alias or the mac address.
This may be useful if, for example, you wish to send cmnds to the group address, but have them ignored by some Tasmota.
e.g. you may set
`BLEAlias 112233445566=livingroomvalve`
on a Tasmota close to the livingroom, but not on other Tasmota, such that
`cmnd/grptopic/EQ3/livingroom/setdaynight with payload "22 17.5"`
is only actioned by that Tasmota.



## EQ3 radiator valve driver

`xdrv_49_BLE_EQ3_TRV` is a driver based on `xdrv_47_BLE_ESP32` for controlling the Bluetooth version of the EQ3 radiator valves.

To enable the driver, you must have the following:

```
#define USE_BLE_ESP32                             // Add support for ESP32 as a BLE-bridge
#define USE_EQ3_ESP32                               // Add support for EQ3 TRV radiator valves - 
```

The EQ3 valves can be controlled and polled via BLE

To list devices and/or see pairing, BLE Active scan must be enabled (command "BLEScan 1", or from configuration).

You MAY use Aliases using `BLEAlias` command in place of MAC address.

The driver recognises the following BLE names as EQ3 devices:

* "CC-RT-BLE"
* "CC-RT-BLE-EQ"
* "CC-RT-M-BLE"

### Commands:

Command|Parameters and description
:---|:---
trv reset|Sets retries to zero and finishes the current command
trv devlist|Reports seen devices. Active scanning required, not passive, as it looks for names
trv scan|Same as devlist
trv MAC state|Reports general state (see below for MQTT)
trv MAC raw|Sends a raw command (in hexadecimal)
trv MAC on|Sets temp to 30. Displays ON on EQ3
trv MAC off|Sets temp to 4.5. Display OFF on EQ3
trv MAC boost|Turns on boost
trv MAC unboost|Turn off boost
trv MAC lock|Locks physical buttons
trv MAC unlock|Unlock physical buttons
trv MAC auto|Sets EQ3 to auto mode
trv MAC manual|Sets EQ3 to manual mode
trv MAC eco|Sets EQ3 to eco mode
trv MAC day|Sets EQ3 to day temp
trv MAC night|Sets EQ3 to night temp
trv MAC settemp 20.5|Sets EQ3 to temp
trv MAC settime|Sets time to Tasmota time (untested)
trv MAC settime|Sets time (hex as per esp32_mqtt_eq3)
trv MAC offset 1.5|Sets offset temp
trv MAC setdaynight 22 17.5|Sets day and night mode temps
trv MAC setwindowtempdur 12.5 30|Set window open temp and duration in mins
trv MAC reqprofile <0-6>|Recall a profile for a day of the week.
trv MAC setprofile <0-6> 20.5-07:30,17-17:00,22.5-22:00,17-24:00|Define a profile with up to 7 temp-HH:MM)

**Barbudor Note : Day of week 0-6 : which day is 0 ? Sunday or Monday ?**

Commands may also be sent over MQTT in the normal way, but in addition to:
```
cmnd/%topic%/EQ3/MAC|Alias/cmd params
```

e.g.
```
cmnd/MyTasmota/EQ3/001A22092EE0/settemp with payload "22.5"
```
or
```
cmnd/MyTasmota/EQ3/livingroom/setdaynight with payload "22 17.5"
```
**Barbudor Note: Above is not clear. Can you give exemples ?**

### Responses to trv commands

In response to the commands above, you will get an immediate response:
```
MQT: stat/%topic%/RESULT = {"trv":"Done|queued|ignoredbusy|invcmd|cmdfail|invidx"}
```
* `Done`: The command was of a type which provides an immediate result.
* `Queued`: The command will be processed by BLE shortly.
* `Ignorebusy`: You may only have one BLE command in progress at a time.
* `Cmdfail`: Failed to queue the command. Maybe other current operations have filled the BLE queu or other fatal failure.
* `Invidx`: Invalid index. All the commands above are equivalent to `trv1`. You would get this if you use `trv0` or `trv2` explicitly.

**Barbudor Note : Invalid index should return "Command Error" as per tasmota standard**

### Results of TRV commands:

Results are sent on MQTT to `stat/%topic%/EQ3/MAC` such as:
```
MQT: stat/%topic%/EQ3/001A22092CDB = {"trv":"00:1a:22:09:2c:db","blestate":"FAILCONNECT","retriesremain":3}
```

Normal message:
```
tele/tasmota_E89E98/EQ3/001A22092EE0 = {
  "trv":"00:1a:22:09:2e:e0",
  "blestate":DONENOTIFIED, - state of the command - FAILxxx | DONExxxx
  "raw":"02010900042C", - raw response in hex
  "temp":22.0, - temp currently set (NOT measured temp)
  "posn":0, - position of the valve (0-100);
  "mode":"manual", 
  "boost":"inactive",
  "dst":"set", - daylight savings time?
  "window":"closed",
  "state":"unlocked",
  "battery":"GOOD"
}
```

Holiday message: as above, but adds `"holidayend":"YY-MM-DD HH:MM"`.

On `trv MAC reqprofile` command, response is :
```
tele/tasmota_E89E98/EQ3 = {"trv":"00:1a:22:09:2e:e0","blestate":DONENOTIFIED,"raw":"02010900042C", "profiledayN":"20.5-07:30,17.0-17:00,22.5-22:00,17.0-24:00"}
```
where N is the day of week (0-6).

On `trv MAC setprofile` command, response is:
```
tele/tasmota_E89E98/EQ3 = {"trv":"00:1a:22:09:2e:e0","blestate":DONENOTIFIED,"raw":"02010900042C","profiledayset":N}
```
where N is the day of week (0-6).

On error, response is
```
tele/tasmota_E89E98/EQ3 = {"trv":"00:1a:22:09:2e:e0","blestate":"FAILxxxx","retriesremain":<1-3>}
```

The driver will try a command three times. When retries are exhausted, response is:
```
tele/tasmota_E89E98/EQ3 = {"trv":"00:1a:22:09:2e:e0","blestate":"FAILxxxx"}
```
