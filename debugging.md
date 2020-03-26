# Debugging sensor restart issue.
The sturgeon platform has an issue that when you stop a sensor and the start it again. It will not output any data.

The sturgeon platform uses the `/system/bin/shd` binary to communicate with the mcu for the sensors.



## Debugging on Android Wear

When you need TWRP:
https://forum.xda-developers.com/smartwatch/huawei-watch/guide-return-to-stock-huawei-watch-t3219596

Installed tools:
- su (https://forum.xda-developers.com/smartwatch/g-watch/tutorial-root-android-wear-6-0-1-t3320215)
- nano (https://forum.xda-developers.com/showthread.php?t=2239421)
- strace (https://raw.githubusercontent.com/andrew-d/static-binaries/master/binaries/linux/arm/strace)

Remount system as rw:
```sh
mount -o rw,remount /system
```

Install su:
```sh
# Make sure adb debugging is enabled.
adb reboot bootloader
# Boot TWRP
fastboot boot TWRP2.8.7.0-Sturgeon.img
# Select Advanced->ADB Sideload from TWRP.
adb sideload wearSU267.zip
```

Install nano:
```sh
# Make sure adb debugging is enabled.
adb reboot bootloader
# Boot TWRP
fastboot boot TWRP2.8.7.0-Sturgeon.img
# Select Advanced->ADB Sideload from TWRP.
adb sideload UPDATE-nano.Terminal.Editor.v4.8-signed.zip
```

Install strace:
```sh
# From PC
adb push strace /sdcard/

# From Android adb shell
su
mount -o rw,remount /system
cd /system/bin/
cp /sdcard/strace .
# Yes 6777, for socket support
chmod 6777 strace

```


## Debugging on AsteroidOS

Make sure you have strace installed.

Edit: `asteroid/src/meta-sturgeon-hybris/conf/machine/sturgeon.conf`
```sh
....
IMAGE_INSTALL += "msm-fb-refresher brcm-patchram-plus iproute2 wpa-supplicant underclock asteroid-hrm strace"
...
```


## SHC tool
We have a handy tool on sturgeon that allows for some debugging.
This tool connects to the sensor daemon and allows to activate/deactivate specific sensors.

You need `root` to use this tool!

Most of this information is taken by experimenting and disassemly of the `/system/bin/shc` binary. Some of this information may not be accurate.


Commands always have two arguments or more:
```
/system/bin/shc <COMMAND> <SENSOR>
/system/bin/shc data gesture
```

The table below shows some of the possible sensors(not all tested or confirmed):

| name | notes |
|------|-------|
| acc  ||
| gravity ||
| linear||
| gesture| wrist gesture sensor|
| step ||
| heart ||
| health ||
| stepdetector ||
| rotation ||
| gamerotation ||


The following table shows some commands that maybe used:
| command | notes |
|---------|-------|
| active  | |
| deactive | |
| data  ||
| version | Returns mcu firmware version |
| info | Does not return anything |
| flush | |
| batch ||
| debug | Cant find debug information |
| selftest ||
| setpa | Unknown |
| setlog | requires a port(?) and level(?) |
| clibration ||
| setdate ||
| sethealth ||
| setmcurawdata | |

## SHD

The deamon responsible for loading the mcu firmware and communication with the Android Backend.

Refers to itself as `sensorhub daemon`.

It expects the firmware to be located at `/system/bin/mcu.bin`.
Creates two sockets: `/dev/socket/shdctl` and `/dev/socket/shddata`.

Log locations from code(are empty thoug...):
- /data/mcudumplog/
- /data/logdata/
- /data/logdata/sensor/
- /data/logdata/sensor/alg.txt
- /data/logdata/sensor/hrm.txt

Uses `/dev/ttyHSL1` for serial communication.

Also refers to /nvdata/ probably used to store calibration info.
`/nvdata/accclibration` and `/nvdata/BARO` are referenced to.

### SHD properties

These are mentioned in the code:
- persist.huawei.shd.debug
- persist.huawei.shd.mculog
- persist.huawei.shd.mcureboot
- persist.huawei.shd.rawdatatype
### How to read from a sensor

```
/system/bin/shc active gesture
/system/bin/shc data gesture
```

## The issue

The issues arises when you execute the following commands:

```sh
# Assumes that the gesture sensor was not active yet.
/system/bin/shc active gesture
/system/bin/shc data gesture
# Do wrist gesture, observe incoming data.

# Restart sensor.
/system/bin/shc deactive gesture
/system/bin/shc active gesture
/system/bin/shc data gesture
# Do wrist gesture, no more incoming data...
```

This issue also appear on Android Wear when manually starting `shd`.
The issue does not occur when `shd` is started from it's `init.sturgeon.rc` script on Android Wear:

```sh
service sensorhd /system/bin/shd
    class main
    socket shdctl stream 0666 root root
    socket shddata stream 0666 root root
    user  root
    group root
    oneshot
```

The `init.rc` script for `shd` on AsteroidOS:
```sh
service sensorhd /system/bin/shd
    class core
```

Adding:
```sh
    socket shdctl stream 0666 root root
    socket shddata stream 0666 root root
    user  root
    group root
    oneshot
```
Does not fix the issue...




## Possible causes of issue.

Currently uknown, but there are some things that might be possible:
- SELinux is enabled on Android Wear
- Android Wear does some additional things that sensorfw does not.