Bluetooth in Tasmota consists of:

## ESP8266 or ESP32 via HM-1x or nRF24L01(+)
This allows for the receiving of BLE advertisments from BLE devices, including "iBeacons"
For this style, `#undef USE_BLE_ESP32`

## ESP32 native Bluetooth Low Energy support
This allows for the receiving of BLE advertisments from BLE devices, including "iBeacons" and BLE sensors, but also for the control of simple BLE devices, providing for reading, writing and receiving notifications. 
For this style, #define USE_BLE_ESP32 
Be aware, enabling of the native BLE on ESP32 has an impact on wifi performance.  Although later SDK helped a bit, expect more lag on the web interface and on MQTT.
If only controlling BLE devices, then scanning can be disabled, wnhich will minimise wifi impact. 

!!! info "Presence detection with iBeacons or BLE sensor gateway using HM-1x or nRF24L01(+) peripherals"

## iBeacon  

!!! info "This feature is included only in tasmota-sensors.bin"
Otherwise you must [compile your build](Compile-your-build). Add the following to `user_config_override.h`:

```
#ifndef USE_IBEACON
#define USE_IBEACON          // Add support for bluetooth LE passive scan of ibeacon devices 
#endif
```
----
  
Tasmota uses a BLE 4.x module to scan for [iBeacon](https://en.wikipedia.org/wiki/IBeacon) devices. This driver is working with [HM-10 and clones](HM-10) and [HM16/HM17](HM-17) Bluetooth modules and potentially with other HM-1x modules depending on firmware capabilities.

### Using ESP32 built-in Bluetooth

You must [compile your build](Compile-your-build) for the ESP32 (since v9.1). Change the following to `user_config_override.h`:

```
#ifdef ESP32
  #define USE_BLE_ESP32
  #define USE_IBEACON_ESP32    // Use internal ESP32 Bluetooth module
#endif // ESP32
```

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

!!! tip
    If using an extenral module, When first connected some modules will be in peripheral mode. You have to change it to central mode using commands `Sensor52 1` and `Sensor52 2`.

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
  
  
  
  
## Tasmota and BLE-sensors

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
  
The LYWSD03MMC, MHO-C401 and the MJYD2S will start to send advertisements with encrypted sensor data after pairing it with the official Xiaomi app. Out-of-the-box the sensors do only publish a static advertisement.  
It is possible to do a pairing and get the necessary decryption key ("bind_key") here: https://atc1441.github.io/TelinkFlasher.html  
This project also provides a custom firmware for the LYWSD03MMC, which then becomes an ATC and is supported by Tasmota too. Default ATC-setting will drain the battery more than stock firmware, because of very frequent data sending.  
This key and the corresponding MAC of the sensor can be injected with the NRFKEY-command (or NRFMJYD2S). It is probably a good idea to save the whole config as RULE like that:  
  
```haskell
rule1 on System#Boot do backlog NRFkey 00112233445566778899AABBCCDDEEFF112233445566; NRFkey 00112233445566778899AABBCCDDEEFF112233445566; NRFPage 6; NRFUse 0; NRFUse 4 endon
```  
(key for two sensors, 6 sensors per page in the WebUI, turn off all sensors, turn on LYWS03)  

(note: for the native ESP32 MI32 driver, the key command is MI32Key, not NRFkey)

LYWSD03MMC sends encrypted sensor data every 10 minutes. As there are no confirmed reports about correct battery presentation of the sensor (always shows 99%), this function is currently not supported.  
MJYD2S sends motion detection events and 2 discrete illuminance levels (1 lux or 100 lux for a dark or bright environment). Additionally battery level and contiguous time without motion in discrete growing steps (no motion time = NMT).    
 
  
#### Working principle of Tasmota BLE drivers (>8.5.)
  
The idea is to provide drivers with as many automatic functions as possible. Besides the hardware setup, there are zero or very few things to configure.  
The sensor namings are based on the original sensor names and shortened if appropriate (Flower care -> Flora). A part of the MAC will be added to the name as a suffix.  
All sensors are treated as if they are physically connected to the ESP8266 device. For motion and remote control sensors MQTT-messages will be published in (nearly) real time.
The ESP32 and the HM-1x-modules are real BLE devices whereas the NRF24L01 (+) is only a generic 2.4 GHz transceiver with very limited capabilities.  

With the USE_BLE_ESP32 define set, the ESP32 Bluetooth allows for both iBeacon and Sensors to be used simultaneously in one build.
It also allows for some basic BLE configuration and veiwing through the web interface.


##### Options to read out the LYWSD03MMC  
  
1. Generate a bind_key  
The web-tool https://atc1441.github.io/TelinkFlasher.html allows the generation of a so-called bind_key by faking a pairing with the Xiaomi cloud. You can copy-paste this key and add the MAC to use this resultig key-MAC-string with key-command (NRFkey or MI32key). Then the driver will receive the sensors data roughly every 10 minutes (in two chunks for humidity and temperature with about a minute in between) and decode the data. This is the most energy efficient way. 
The current way of storing these keys on the ESP32 is to use RULES like that (for the NRF24L01 you would use NRFkey):  
```haskell
rule1 on System#Boot do backlog MI32key 00112233445566778899AABBCCDDEEFF112233445566; MI32key 00112233445566778899AABBCCDDEEFFAABBCCDDEEFF endon
```  
This option is currently not available for the HM-10 because of memory considerations as part of the standard sensor-firmware package.  
  
2. Flash custom ATC-firmware  
Use the same https://atc1441.github.io/TelinkFlasher.html to flash a custom ATC-firmware on the LYWSD03MMC. This will work out of the box with all three Tasmota-drivers. There is a slight chance of bricking the sensor, which would require some soldering and compiling skills to un-brick. This firmware does send data more frequently and is a little bit more power hungry than the stock firmware.  
  
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


## Generic BLE driver - #define USE_BLE_ESP32  
<details>
  <summary>Click to expand Generic BLE driver</summary>

This is the basis for ESP32 native Bluetooth low energy going forwards.
The base level driver provides a consistent mechanism for listening for advertisments, and for simple accesses to BLE devices.
It provides the ability to enable/disable BLE from the web interface (stored in configuration), ability to turn on Active scanning (not stored), and a simple list of observed devices in the web configuration.

It also provides some basic BLE commands and controls which can be used to implement driving BLE devices from external sources - the same capabilities are offered to other drivers for development purposes. 

### Commands:

### BLEDebug
BLEDebug - log more information
BLEDebug1 - log less information (default)

### BLEDevices
BLEDevices/BLEDevices1 - Cause a tele msg with devices
BLEDevices0 - Clear device list.

### BLEMaxAge
The ‘Age’ of a BLE device is the time since an advertisement was last seen.
The BLE driver will remove devices from the displayed list when they hit a maximum age, the default being 600s.
If set to zero, devices will never be forgotten.

BLEMaxAge - display current max age for a BLE device before it is removed (in seconds)
BLEMaxAge seconds - set the Max age in seconds for a BVLE device before it is removed from those listed.

### BLEAdv
BLEADV0 - disable MQTT list of devices
BLEADV1 - (default) enable MQTT list of devices

The list of devices is published on topic 
tele/tasmota_XXXXX/BLE
And looks like:
```
{
  "active":{
    "37E6BCD640AC":{"n":"","r":-58},
    "001A22092C9A":{"n":"CC-RT-M-BLE","r":-87, "a":"MyEQ3"},
    "4C65A8DAF607":{"n":"MJ_HT_V1","r":-73},
    "001A22092CDB":{"n":"CC-RT-M-BLE","r":-86},
    "D6E0138D9201":{"n":"Charge 3","r":-75},
    "4C65A8DAF43A":{"n":"MJ_HT_V1","r":-78},
    "001A22092FB7":{"n":"CC-RT-M-BLE","r":-88},
    "21C59A55A412":{"n":"","r":-52}
  }
}
```

I.e. a JSON where the keys are the MAC address of devices, and each key has a value of a structure containing a key ‘n’, which is the device name, and a key ‘r’ which is the rssi, and a key ‘a’ if an alias exists for this MAC.

### BLEOp
This is the command to use to read/write/get notification from a device.
This functionality allows you to control devices which are not supported by Tasmotas drivers.
BLEOp0 - list operation currently present.

BLEOp1 - set details about an operation and queue it.
BLEOp1 m:MAC|Alias s:svc c:characteristic n:notifychar w:hextowrite u:unique r go
(everything after svc is optional)

BLEOp2 - queue an operation setup with BLEOp1 if you did not state ‘go’

returns: Done|FailCreate|FailNoOp|FailQueue|InvalidIndex|{"opid":opid,"u":unique}

Example:
Write: ‘03’ to 
service 3e135142-654f-9090-134a-a6ff5bb77046
characteristic 3fa4585a-ce4a-3bad-db4b-b8df8179ea09
Unique 1234 (32 bit number)

And wait for a notify from 
d0e8434d-cd29-0996-af41-6c90f4e0eb2a

`BLEOp1 M:001A22092CDB s:3e135142-654f-9090-134a-a6ff5bb77046 c:3fa4585a-ce4a-3bad-db4b-b8df8179ea09 w:03 n:d0e8434d-cd29-0996-af41-6c90f4e0eb2a u:1234 go`

Read:
service 3e135142-654f-9090-134a-a6ff5bb77046
characteristic 3fa4585a-ce4a-3bad-db4b-b8df8179ea09

`BLEOp1 M:001A22092CDB s:3e135142-654f-9090-134a-a6ff5bb77046 c:3fa4585a-ce4a-3bad-db4b-b8df8179ea09 r u:1235 go`

When an BLEOp0 is called, MQTT send one or more:
```
21:43:08 MQT: tele/tasmota_esp32/BLE = {"BLEOperation":{"opid":"0","stat":"1","state":"START","MAC":"001A22092CDB","svc":"3e135142-654f-9090-134a-a6ff5bb77046","char":"3fa4585a-ce4a-3bad-db4b-b8df8179ea09","notifychar":"d0e8434d-cd29-0996-af41-6c90f4e0eb2a","wrote":"03"}}
```
When the operation is done/failed, the data is sent to MQTT again.

A failure may look like:
```
21:43:10 MQT: tele/tasmota_esp32/BLE = {"BLEOperation":{"opid":"0","stat":"-11","state":"FAILCONNECT","MAC":"001A22092CDB","svc":"3e135142-654f-9090-134a-a6ff5bb77046","char":"3fa4585a-ce4a-3bad-db4b-b8df8179ea09","notifychar":"d0e8434d-cd29-0996-af41-6c90f4e0eb2a","wrote":"03"}}
```

A success:
```
21:43:22 MQT: tele/tasmota_esp32/BLE = {"BLEOperation":{"opid":"1","stat":"7","state":"NOTIFIED","MAC":"001A22092CDB","svc":"3e135142-654f-9090-134a-a6ff5bb77046","char":"3fa4585a-ce4a-3bad-db4b-b8df8179ea09","notifychar":"d0e8434d-cd29-0996-af41-6c90f4e0eb2a","wrote":"03","notify":"020109000429"}}
```
### BLEMode
BLEMode0 -> kill BLE completely
BLEMode1 -> start BLE, but don’t scan unless asked to
BLEMode2 -> (default) start BLE, do regular 20s scans

### BLEDetails
BLEDetails0 -> (default) don’t send me anything
BLEDetails1 MAC -> send me details for mac once
BLEDetails2 MAC -> send me details for mac every advert if possible
BLEDetails3 -> send me details for ALL macs every advert if possible

MQTT sends look like:
```
MQT: tele/tasmota_esp32/BLE = {
  "details":{
    "mac":"001A22092C9A","p":"0C0943432D52542D4D2D424C450CFF0000000000000000000000"
  }
}
```

### BLEScan
BLEScan0 0 -> (default) Scans are passive
BLEScan0 1 -> Scans are active
BLEScan/BLEScan1 -> do a scan now if BLEMode1
BLEScan/BLEScan1 timesec -> do a scan now if BLEMode1 for timesec seconds

### BLEAlias
BLEAlias mac=name mac=name

Adds an alias for mac
The alias may be used in place of a mac address for any command.

BLEAlias2 - clear all aliases.


## Usage notes

BLEMode1:
If you use BLEMode1, then both iBeacon and MI32 will stop receiving adverts (unless you manually trigger a scan with BLEScan0).

However, you CAN still send BLEOp commands, and expect it to connect and operate.

BLEScan1 0/1:
I have not investigated in any depth as to the effect of using active vs passive scans.
Note that for some devices, the advertisement data received is different for passive and active mode.

BLEOp:
It will fail to add an operation to the queue if 10 are already queued.  However, the queue management could be enhanced, so probably best to serialise any operations from a driver (e.g. the MI32 driver uses Operations to read the battery level on some sensors.  It queues one operation, and waits for that to finish before queuing another).
This leaves room for both other drivers, and also user operations to be queued.
If an external application is adding operations, the same should be applied - queue one, and wait for it to succeed or fail before adding another.
We could reasonably expect the driver to work with up to 6 operations in progress - but if one source of operations steals all 6 slots at once, it will affect others who may be wanting to perform operations.

Normal operations against BLE devices will be very variable with regard to the time taken.  E.g. with the connection timeout set to 30s, you can expect an operation against a device which cannot be reached to take 30s.
Since all operations are performed serially, any single operation may take up to 3 minutes to fail (e.g. if there were 6 in progress).  

We COULD alleviate this by running multiple BLE clients at the same time.  The driver has been designed so that this could be possible in the future, but first we must establish it’s true stability in use.

## Creating a driver which uses BLE
xdrv_47_BLE_ESP32 is designed to be used by other drivers which need to use BLE advertisements or interact with BLE devices.

Two drivers that use BLE_ESP32 are:
tasmota\xsns_62_MI_ESP32_BLE_ESP32.ino
tasmota\xsns_52_ibeacon_BLE_ESP32.ino

Please make your driver depends upon 

`#ifdef USE_BLE_ESP32`

Include the file 
`#include "xdrv_47_BLE_ESP32.h"`

All functions and constants are in namespace BLE_ESP32::

### ‘Hearing’ advertisements:

Create a function like:

```
int MyDriverNameAdvertismentCallback(BLE_ESP32::ble_advertisment_t *pStruct)
{
  // the NimBLE advertisement...
  BLEAdvertisedDevice *advertisedDevice = pStruct->advertisedDevice;

  // some things are already extracted:
  int RSSI = pStruct->RSSI;
  const uint8_t *addr = pStruct->addr;
  const char *name = pStruct->name;

  AddLog_P(LOG_LEVEL_DEBUG,PSTR("Adv received for %s(%s)"),
    ((std::string)advertisedDevice->getAddress()).c_str(), name);
  return 0;
}
```
From the callback, or anything you call from the callback, don’t do anything directly with Tasmota globals, or call any MQTT functions or standard log functions!

Register this callback with 

`BLE_ESP32::registerForAdvertismentCallbacks("MyDriverName",  MyDriverNameAdvertismentCallback);`



Performing Read, Write on a device, and/or requesting the next Notify:

The BLE operations are controlled by:

A structure:
`BLE_ESP32::generic_sensor_t *op = nullptr;`

Functions:
```
  // ALWAYS use this function to create a new one.
  int res = BLE_ESP32::newOperation(&op);

  // Queue with this.
  int res = BLE_ESP32::extQueueOperation(&op);
  if (!res){
    // if it fails to add to the queue, do please delete it
    BLE_ESP32::freeOperation(&op);
    AddLog_P(LOG_LEVEL_ERROR,PSTR("Failed to queue new operation - deleted"));
  }
```

Operations are added to a limited queue, and performed serially.
BLE is quite SLOW, so you can expect some seconds before a response.
Also, we don’t hurry things.  Operations are only transferred from the queue to the active 
Operation once per second.  Similarly, completed operations are only examine once per seconds.

Whilst an operation is in progress, Adverts will stop coming in.

Operations have a status, +ve -> in progress or success. -ve -> failed.
You can get the name of a status from BLE_ESP32::getStateString(state);
States are from the set defined in the .h file GEN_STATE_XXXX





A generic function for running an operation may be:
```
int genericOpCompleteFn(BLE_ESP32::generic_sensor_t *op){
  if (op->state <= GEN_STATE_FAILED){
    AddLog_P(LOG_LEVEL_ERROR,PSTR("Operation failed with state %s", 
       BLE_ESP32::getStateString(op->state));
    return 0;
  }

  //Do something with your shiny data which was read 
  If (op->readlen){
   If (!op->readtruncated){
   }
  }
  //Do something with your shiny data which was notified 
  If (op->notifylen){
   If (!op->notifytruncated){
   }
  }


  // note: the op will be deleted automatically once all calls are over.
  return 0;
}


int myspecialdatamodifyer(BLE_ESP32::generic_sensor_t *op){
  // change the data to write based on the data read.
}



#define OPTYPE_READ 0
#define OPTYPE_READMODIFYWRITE 1


bool BLEOperation( utin8_t *addr, int optype, const char *svc, const char *charactistic, 
  const char *notifychar = nullptr, const uint8_t *data = nullptr, int datalen = 0) {
  
  if (!svc || !svc[0]){
    return 0;
  }

  BLE_ESP32::generic_sensor_t *op = nullptr;

  // ALWAYS use this function to create a new one.
  int res = BLE_ESP32::newOperation(&op);
  if (!res){
    AddLog_P(LOG_LEVEL_ERROR,PSTR("Can't get a newOperation from BLE"));
    return 0;
  } else {
    AddLog_P(LOG_LEVEL_DEBUG,PSTR("got a newOperation from BLE"));
  }

  op->addr = NimBLEAddress(addr);

  bool havechar = false;
  op->serviceUUID = NimBLEUUID(svc);
  if (charactistic && charactistic[0]){
    havechar = true;
    op->characteristicUUID = NimBLEUUID(charactistic);
  }
  if (notifychar && notifychar[0]){
    op->notificationCharacteristicUUID = NimBLEUUID(notifychar);
  }    

  // if we have data, assume this is a write operation
  if (data && datalen) {
    op->writelen = datalen;
    memcpy(op->dataToWrite, data, datalen);
  } else {
    // else if we have a RW characteristic, must be a read operation
    if (!datalen && havechar){
      op->readlen = 1; // if we don't set readlen, then it won't read
    }
  }

  // the only times we intercept between read and write
  if (optype == OPTYPE_READMODIFYWRITE){
    op->readlen = 1; // if we don't set readlen, then it won't read
    op->readmodifywritecallback = (void *)myspecialdatamodifyer;
  }

  // this op will call us back on complete or failure.
  op->completecallback = (void *)genericOpCompleteFn;

  // you can use context to indicate anything you like.
  uint32_t context = 0; 
  op->context = (void *)context;

  res = BLE_ESP32::extQueueOperation(&op);
  if (!res){
    // if it fails to add to the queue, do please delete it
    BLE_ESP32::freeOperation(&op);
    AddLog_P(LOG_LEVEL_ERROR,PSTR("Failed to queue new operation - deleted"));
  }

  return res;
}
```

### Other callbacks:

You can register to know when a scan ends, and also for ALL operations:
void registerForOpCallbacks(const char *tag, BLE_ESP32::OPCOMPLETE_CALLBACK* pFn);
void registerForScanCallbacks(const char *tag, BLE_ESP32::SCANCOMPLETE_CALLBACK* pFn);

These are not normally required for a driver.


### Other functions:
A temporary safe logging mechanism.  This has a max of 40 chars, and a max of 30 slots per 50ms.
This will cause logs to be written if called from a thread other than the main thread/task, but not cause problems with corruption.  
int BLE_ESP32::SafeAddLog_P(uint32_t loglevel, PGM_P formatP, ...);


Interpreting an address supplied by the user:
Please use this function.
int BLE_ESP32::getAddr(uint8_t *dest, char *src);
Returns 0 for invalid/not found, 1 for MAC parsed, 2 for Alias found.
This will interpret AABBCCDDEEFF or AA:BB:CC:DD:EE:FF or an alias set with BLEAlias.
dest must be `uint8_t addr[6];`
Establishing if a device is still present, or get it’s age:
Please use this function.
int BLE_ESP32::devicePresent(uint8_t*mac);
Where mac is the BINARY mac address, array of 6 uint8_t
Returns 0 for the device having timed out, or age in seconds.
</details>


## EQ3 radiator valve driver
<details>
  <summary>Click to expand xdrv_49_BLE_EQ3_TRV</summary>

This driver is for controlling the Bluetooth version of the EQ3 radiator valves.

To enable the driver, you must have the following:

```
#define USE_BLE_ESP32                             // Add support for ESP32 as a BLE-bridge
#define USE_EQ3_ESP32                               // Add support for EQ3 TRV radiator valves - 
```

The EQ3 valves can be controlled and polled via BLE

To list devices and/or see pairing, BLE Active scan must be enabled (command "BLEScan 1", or from configuration).

You MAY use Aliases if set using BLE commands in place of MAC address.

The driver recognises the following BLE names as EQ3 devices:
"CC-RT-BLE"
"CC-RT-BLE-EQ"
"CC-RT-M-BLE"

### Commands:

trv reset - sets retries to zero and finishes the current command
trv devlist - report seen devices.  Active scanning required, not passive, as it looks for names
trv scan - same as devlist
trv MAC state - report general state (see below for MQTT)
trv MAC raw (hex to send) - send a raw command
trv MAC on - set temp to 30 -> display ON on EQ3
trv MAC off - set temp to 4.5 -> display OFF on EQ3
trv MAC boost - set boost
trv MAC unboost - turn off boost
trv MAC lock - manual lock of physical buttons
trv MAC unlock - manual unlock of physical buttons
trv MAC auto - set EQ3 to auto mode
trv MAC manual - set EQ3 to manual mode
trv MAC eco - set EQ3 to eco mode?
trv MAC day - set EQ3 to day temp
trv MAC night - set EQ3 to night temp
trv MAC settemp 20.5 - set EQ3 to temp
trv MAC settime - set time to Tasmota time (untested)
trv MAC settime (hex as per esp32_mqtt_eq3) - set time
trv MAC offset 1.5 - set offset temp
trv MAC setdaynight 22 17.5 - set day and night mode temps
trv MAC setwindowtempdur 12.5 30 - set window open temp and duration in mins

trv MAC reqprofile <0-6> - request a profile for a day fo the week.
trv MAC setprofile <0-6> 20.5-07:30,17-17:00,22.5-22:00,17-24:00 (up to 7 temp-HH:MM) - set a profile for a day of the week.

Commands may also be sent over MQTT in the normal way, but in addition to:

cmnd/tasmota_E89E98/EQ3/MAC|Alias/cmd payload=params


### Responses:
In response to the commands above, you will get an immediate response:
`MQT: stat/tasmota_esp32/RESULT = {"trv":"Done|queued|ignoredbusy|invcmd|cmdfail|invidx"}`

Done - the command was of a type which provides an immediate result.
Queued - the command will be processed by BLE shortly.
Ignorebusy - You may only have one BLE command in progress at a time.
Cmdfail - failed to queue the command - maybe other things have ALL available BLE queued commands? - or other fatal failure.
Invidx - all the commands above are ‘trv1’, so you would get this if you use trv0 or trv2 explicitly.


### Results of BLE commands:

Results are sent on MQTT to:
stat/tasmota_esp32/EQ3/MAC
e.g.
`MQT: stat/tasmota_esp32/EQ3/001A22092CDB = {"trv":"00:1a:22:09:2c:db","blestate":"FAILCONNECT","retriesremain":3}`

normal:
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
``

holiday:
as above, but adds ,"holidayend":"YY-MM-DD HH:MM"

when trv mac reqprofile is used:
`tele/tasmota_E89E98/EQ3 = {"trv":"00:1a:22:09:2e:e0","blestate":DONENOTIFIED,"raw":"02010900042C", "profiledayN":"20.5-07:30,17.0-17:00,22.5-22:00,17.0-24:00"}`
where N is the day (0-6).

when trv mac setprofile is used:
`tele/tasmota_E89E98/EQ3 = {"trv":"00:1a:22:09:2e:e0","blestate":DONENOTIFIED,"raw":"02010900042C","profiledayset":N}`
where N is the day (0-6).

on error:
`tele/tasmota_E89E98/EQ3 = {"trv":"00:1a:22:09:2e:e0","blestate":"FAILxxxx","retriesremain":<1-3>}`
when retries exhausted:
`tele/tasmota_E89E98/EQ3 = {"trv":"00:1a:22:09:2e:e0","blestate":"FAILxxxx"}`

The driver will try a command three times.
</details>
