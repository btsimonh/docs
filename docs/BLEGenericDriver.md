Bluetooth Low Energy in Tasmota on ESP32 using the native radio:

## Enabling the driver

You must be building for ESP32 - ESP8266 has no native BLE capability.

To enable the driver, this must be defined:

`#define USE_BLE_ESP32` 

Be aware, enabling of the native BLE on ESP32 has an impact on wifi performance.  Although later SDK helped a bit, expect more lag on the web interface and on MQTT.
If only controlling BLE devices, then scanning can be disabled, which will minimise wifi impact.

This is compiled by default in the Sensors firmware, but you still need to enable BLE using the web interface `configure BLE` button or setoption115 1.

### Driver features

When the BLE_ESP32 driver is enabled and BLE is enabled, Tasmota will by default scan for nearby BLE devices every 20s.

ALL BLE devices are noted and displayed in the Configure BLE page.  To see an updaqted list, you must refresh the page.

Devices you may 'find' include BLE sensors (e.g. the 'MI' range), fitness watches, iBeacons, Toothbrushes, Scales, etc.
You will also see some random MAC addresses appearing - these are Mobile phones which use a random MAC address when advertiseing - for example they regularly advertise Covid contact tracing information.  Unfortunately, there is no easy way to exclude these, but indeed the BLE driver enables you to catch some of the contact tracing information.

Scanning can be configued to be 'passive' (default), or 'Active'.  In 'Active' mode, tasmota queries the dveices for some additional information (e.g. the name of the device)

The driver also allows you to perform simple active operations, like reading a characteristic, writing a characteristic, and registering for a notification.  When performing these operaitons, the BLE_ESP32 driver connects to the device, performs the operation, and then disconnects.  Operations can take between 1 and 10 seconds, and BLE is not necessarily reliable, so expect failures.



### BLE commands summary:

Command|Parameters
:---|:---
BLEMode<a class="cmnd" id="blemode"></a>|Change the operational mode of the BLE driver.<BR>`BLEMode0` = disable regular BLE scans.<BR>`BLEMode1` = BLE scan on command only.<BR>`BLEMode2` = regular BLE scanning (default).
BLEScan<a class="cmnd" id="blescan"></a>|Cause/Configure BLE a scan<BR>`BLEScan0 0..1` = enable or disable Active scanning. (an active scan will gather extra data from devices, including name)<BR>`BLEScan` = Trigger a 20s scan now if in BLEMode1<BR>`BLEScan n` = Trigger a scan now for n seconds if in BLEMode1
BLEDetails<a class="cmnd" id="bledetails"></a>|Display details about recevied adverts<BR>`BLEDetails0` = disable showing of details.<BR>`BLEDetails1 mac|alias` = show the next advert from device mac|alias<BR>`BLEDetails2 mac|alias` = show all advert from device mac|alias (some may be lost).<BR>`BLEDetails3` = show all adverts from all devices (some will be lost).
BLEAlias<a class="cmnd" id="blealias"></a>|Set Alias names for devices.  A device may be referred to by it's alias in subsequent commands<BR>`BLEAlias mac=alias mac=alias ...` = set one or more aliases from devices.<BR>`BLEAlias2` = clear all aliases.
BLEName<a class="cmnd" id="blename"></a>|Read or write the name of a BLE device.<BR>`BLEName mac|alias` = read the name of a device using 1800/2A00.<BR>`BLEName mac|alias` = write the name of a device using 1800/2A00 - many devices are read only.
BLEDevices<a class="cmnd" id="bledevices"></a>|Cause a list of known devices to be sent on MQTT, or Empty the list of known devices.<BR>`BLEDevices0` = clear the known devices list.<BR>`BLEDevices` = Cause the known devices list to be published on stat/TASName/BLE.
BLEMaxAge<a class="cmnd" id="blemaxage"></a>|Set the timeout for device adverts.<BR>`BLEMaxAge n` = set the devices timeout to n seconds.<BR>`BLEMaxAge` = display the device timeout.
BLEOp<a class="cmnd" id="bleop"></a>|Perform a simple active BLE operation (read/write/notify).<BR>`BLEOp0` = trigger publish of operations in progress to MQTT at stat/TASName/BLE.<BR>`BLEOp (parameters)` = queue a BLE operation<BR>Paramaters:0<BR>m:mac|alias` - the device to operate on<BR>`s:uuid` - the service to use<BR>`c:uuid` - the read or write characteristic to use<BR>`n:uuid` - the notify characteristic to use register for<BR>`w:hexdata` - data to write in hex<BR>`r` - perform a read<BR>`u:number` - a unique reference to recognise responses by<BR>`go` - start the operation.<BR>example: `BLEOp m:A4C1386A1E24 s:180f c:2a19 r go`
BLEDebug<a class="cmnd" id="bleop"></a>|Set BLE debug level.<BR>`BLEDebug` = show extra debug information<BR>`BLEDebug0` = suppress extra debug


### BLE commands detailed description

#### BLEOp command
This is the command you would use to perform simple active operations against a device.

Command|Parameters and description
:---|:---
`BLEOp0`|Lists operations currently in queue.
`BLEOp|BLEOp1`|Sets details about an operation and queue it. Full syntax is <br>`BLEOp1 m:MAC|Alias s:svc c:characteristic n:notifychar w:hextowrite u:unique r go`<br>Everything after svc is optional.
`BLEOp2`|Queues an operation setup with `BLEOp1` if you did not state `go`.<br>returns: `Done`, `FailCreate`, `FailNoOp`, `FailQueue`, `InvalidIndex`, `{"opid":opid,"u":unique}`

**Example 1: Write operation**

Write `03` to service `3e135142-654f-9090-134a-a6ff5bb77046` with characteristic `3fa4585a-ce4a-3bad-db4b-b8df8179ea09` and unique `1234` (32 bit number) and wait for a notification from  `d0e8434d-cd29-0996-af41-6c90f4e0eb2a`
```
BLEOp1 M:001A22092CDB s:3e135142-654f-9090-134a-a6ff5bb77046 c:3fa4585a-ce4a-3bad-db4b-b8df8179ea09 w:03 n:d0e8434d-cd29-0996-af41-6c90f4e0eb2a u:1234 go
```

**Example 2: Read operation**

Read from service `3e135142-654f-9090-134a-a6ff5bb77046` with characteristic `3fa4585a-ce4a-3bad-db4b-b8df8179ea09`
```
BLEOp1 M:001A22092CDB s:3e135142-654f-9090-134a-a6ff5bb77046 c:3fa4585a-ce4a-3bad-db4b-b8df8179ea09 r u:1235 go
```

When a `BLEOp0` is sent, the following MQTT is published:
```
MQT: tele/%topic%/BLE = {"BLEOperation":{"opid":"0","stat":"1","state":"START","MAC":"001A22092CDB","svc":"3e135142-654f-9090-134a-a6ff5bb77046","char":"3fa4585a-ce4a-3bad-db4b-b8df8179ea09","notifychar":"d0e8434d-cd29-0996-af41-6c90f4e0eb2a","wrote":"03"}}
```
When the operation is done/failed, the data is sent to MQTT again.

A failure may look like:
```
MQT: tele/%topic%/BLE = {"BLEOperation":{"opid":"0","stat":"-11","state":"FAILCONNECT","MAC":"001A22092CDB","svc":"3e135142-654f-9090-134a-a6ff5bb77046","char":"3fa4585a-ce4a-3bad-db4b-b8df8179ea09","notifychar":"d0e8434d-cd29-0996-af41-6c90f4e0eb2a","wrote":"03"}}
```

A success:
```
MQT: tele/%topic%/BLE = {"BLEOperation":{"opid":"1","stat":"7","state":"NOTIFIED","MAC":"001A22092CDB","svc":"3e135142-654f-9090-134a-a6ff5bb77046","char":"3fa4585a-ce4a-3bad-db4b-b8df8179ea09","notifychar":"d0e8434d-cd29-0996-af41-6c90f4e0eb2a","wrote":"03","notify":"020109000429"}}
```

### JSON on BLE topic
The list of active BLE devices is a JSON object published on `tele/%topic%/BLE`. The JSON object has `active` as the 1st level key. The 2nd level object use the MAC address of devices key. Each sub-object contains the device name as `n`, the RSSI as `r` and optionally the alias of the device as `a`:
```
tele/tasmota_esp32/BLE = {
  "Time":"2021-01-10T19:57:54",
  "BLEDevices":{
    "total":11,
    "001A22092FB7":{"i":0,"n":"CC-RT-M-BLE","r":-86},
    "4C65A8DAF607":{"i":1,"n":"MJ_HT_V1","r":-84},
    "001B66C0CF4C":{"i":2,"n":"HD 450BT","r":-80},    
    "001A22092CDB":{"i":3,"n":"CC-RT-M-BLE","r":-86},
    "A4C1387FC1E1":{"i":4,"n":"LYWSD03MMC","r":-70},
    "001A22092C9A":{"i":5,"n":"CC-RT-M-BLE","r":-87},
    "A4C1386A1E24":{"i":6,"n":"LYWSD03MMC","r":-73},
    "142BC286F794":{"i":7,"r":-53},
    "D6E0138D9201":{"i":8,"n":"Charge 3","r":-87},
    "21BF2683F8A9":{"i":9,"r":-51},
    "10758BBEF8E0":{"i":10,"r":-96}
  }
}
```
if requested via the command `BLEDevices`, it is published on stat/



When `BLEDetails` is enabled, a JSON with `details` as 1st level key is published with the following format:
```
MQT: tele/%topic%/BLE = {
  "details":{
    "mac":"001A22092C9A",
    "p":"0C0943432D52542D4D2D424C450CFF0000000000000000000000",
    "uuid":"servicedata",
    "uuid2":"servicedata",
  }
}
```
if some details were overwritten (2 advrets arrived before the MQTT could be sent), then "lost":true will also appear.


### Usage notes

`BLEMode1`:

  If you use BLEMode1, then both iBeacon and MI32 will stop receiving adverts (unless you manually trigger a scan with BLEScan0).

  However, you CAN still send BLEOp commands, and expect it to connect and operate.

`BLEScan1 0/1`:

  Note that for some devices, the advertisement data received is different for passive and active mode.

`BLEOp`:

  It will fail to add an operation to the queue if 10 are already queued.  However, the queue management could be enhanced, so probably best to serialise any operations from a driver (e.g. the MI32 driver uses Operations to read the battery level on some sensors. It queues one operation, and waits for that to finish before queuing another).
  
  This leaves room for both other drivers, and also user operations to be queued. If an external application is adding operations, the same should be applied - queue one, and wait for it to succeed or fail before adding another. We could reasonably expect the driver to work with up to 6 operations in progress - but if one source of operations steals all 6 slots at once, it will affect others who may be wanting to perform operations.

  Normal operations against BLE devices will be very variable with regard to the time taken. E.g. with the connection timeout set to 30s, you can expect an operation against a device which cannot be reached to take 30s. Since all operations are performed serially, any single operation may take up to 3 minutes to fail (e.g. if there were 6 in progress).  

  We COULD alleviate this by running multiple BLE clients at the same time.  The driver has been designed so that this could be possible in the future, but first we must establish it’s true stability in use.

## High level driver based on BLE_ESP32
`xdrv_47_BLE_ESP32` is designed to be used by other upper-level drivers which need to use BLE advertisements or interact with BLE devices. `xsns_62_MI_ESP32_BLE_ESP32.ino` and `xsns_52_ibeacon_BLE_ESP32.ino` are example of such driver.

Such a driver must depend on BLE_ESP32 feature and code whuld be wrapped within : 
```
#ifdef USE_BLE_ESP32
#ifdef USE_your_driver_name

// your driver code

#endif // #ifdef USE_your_driver_name
#endif // #ifdef USE_BLE_ESP32
```

All functions and constants are in namespace BLE_ESP32::

### Receiving advertisements:

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

The BLE operations are controlled by a structure `BLE_ESP32::generic_sensor_t *op`

Creating a new functions:
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
You can get the name of a status from `BLE_ESP32::getStateString(state)`.
States are defined in the header file as `GEN_STATE_XXXX`.

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
```
void registerForOpCallbacks(const char *tag, BLE_ESP32::OPCOMPLETE_CALLBACK* pFn);
void registerForScanCallbacks(const char *tag, BLE_ESP32::SCANCOMPLETE_CALLBACK* pFn);
```
These are not normally required for a driver.


### Other functions:

#### Interpreting an address supplied by the user:

```
int BLE_ESP32::getAddr(uint8_t *dest, char *src);
```
Where `dest` must be `uint8_t addr[6];`

Returns:

* `0` for invalid/not found, 
* `1` for MAC parsed,
* `2` for Alias found.

This will interpret `AABBCCDDEEFF` or `AA:BB:CC:DD:EE:FF` or an alias set with `BLEAlias`.

#### Establishing if a device is still present, or get it’s age:

```
int BLE_ESP32::devicePresent(uint8_t*mac);
```
Where `mac` is the BINARY mac address as `uint8_t mac[6]`

Returns:

* `0` for the device having timed out, or
* age in seconds.
