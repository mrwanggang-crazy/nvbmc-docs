# Nvidia GPU PCIe card management via NvBMC
This document describes how to use NvBMC on a platform BMC to manager Nvidia GPU
PCIe card. This applies to the following GPUs:
1) B40 (Blackwell)

## Management overview
The following tables lists OOB mamagement features that a platform BMC may
implement to manage B40. It also shows OpenBMC/NvBMC services that provide the
feature.

| Feature               | Protocol              | Transport | NvBMC/OpenBMC repo                                                  |
| --------------------- | --------------------- | --------- | ------------------------------------------------------------------- |
| Inventory             | IPMI FRU EEPROM       | I2C       | [OpenBMC entity-manager](https://github.com/openbmc/entity-manager) |
| Enumeration           | MCTP                  | USB       | [NvBMC libmctp](https://github.com/NVIDIA/libmctp)                  |
| Telemetry             | NSM over MCTP         | USB       | [NvBMC nsmd](https://github.com/NVIDIA/nsmd)                        |
| Firmware Update       | PLDM Type 5 over MCTP | USB       | [NvBMC pldm](https://github.com/NVIDIA/pldm)                        |
| Attestation           | SPDM over MCTP        | USB       | [NvBMC spdm](https://github.com/NVIDIA/spdm)                        |
| Firmware Recovery     | OCP Recovery          | I2C       | [NvBMC pldm](https://github.com/NVIDIA/pldm)                        |
| OOB management API    | Redfish               | HTTP      | [NvBMC bmcweb](https://github.com/NVIDIA/bmcwwb)                    |

NvBMC is based on OpenBMC and the services listed below will have
dependecies on other OpenBMC repos - those are not listed above and can be
determined by looking at the bitbake recipe of the service. A key such
repository is [phosphor-dbus-interfaces]() - this repo may contain interfaces that
don't exist is upstream OpenBMC.

## Single GPU service enablement
This section describes how to enable/configure NvBMC/OpenBMC services for
various OOB features.

### Inventory
Use upstream [OpenBMC entity-manager](https://github.com/openbmc/entity-manager)
to read and recognize the GPU via I2C FRU EEPROM.

#### Step1: Write bitbake recipe

#### Step2: Write Configuration Files
```
B40 IPMI FRU EEPROM I2C address: 0x50 (7-bit), 0xA0 (8-bit)
Entity-manager PROBE statement:
```

#### Step3: Check expected D-Bus tree

### Enumeration
[NvBMC libmctp](https://github.com/NVIDIA/libmctp) can disover the GPU as an
MCTP USB endpoint as well as to perform MCTP Tx/Rx.

#### Step1: Write bitbake recipe

A sample bitbake recipe append that enables NVBMC libmctp to generate a
tunneling daemon and an MCTP control daemon with support for in-kernel MCTP
sockets.

``` bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI = "git://git@gitlab-master.nvidia.com:12051/dgx/bmc/libmctp.git;protocol=ssh;branch=develop \
           file://default"
SRCREV = "0f62812a5cdd2facbea804539cc568d48b426266"

inherit obmc-phosphor-dbus-service obmc-phosphor-systemd

DEPENDS += "json-c \
            i2c-tools \
            libusb1 \
           "

SYSTEMD_SERVICE:${PN}:remove = " mctp-spi-ctrl.service \
                                 mctp-spi-demux.service \
                                 mctp-spi-demux.socket \
                                 mctp-pcie-ctrl.service  \
                                 mctp-pcie-demux.service \
                                 mctp-pcie-demux.socket  \
                               "

SYSTEMD_SERVICE:${PN} = "mctp-usb-tun.service \
                         mctp-usb-ctrl.service \
                        "

CONFFILES:${PN} = "${datadir}/mctp/mctp"

FILES:${PN}:append = "${datadir} ${datadir}/mctp"

do_install:append() {
    install -d ${D}${datadir}/mctp
    install -m 0644 ${WORKDIR}/mctp ${D}${datadir}/mctp/mctp
    rm -f ${D}${nonarch_base_libdir}/systemd/system/mctp-pcie-ctrl.service
    rm -f ${D}${nonarch_base_libdir}/systemd/system/mctp-pcie-demux.service
    rm -f ${D}${nonarch_base_libdir}/systemd/system/mctp-pcie-demux.socket
    rm -f ${D}${nonarch_base_libdir}/systemd/system/mctp-spi-ctrl.service
    rm -f ${D}${nonarch_base_libdir}/systemd/system/mctp-spi-demux.service
    rm -f ${D}${nonarch_base_libdir}/systemd/system/mctp-spi-demux.socket
}

EXTRA_OEMESON += " -Denable-usb=enabled -Dmctp-in-kernel-enable=enabled -Denable-mctp-tun=enabled "
```

#### Step2: Write Configuration Files

The tunneling daemon only supports one MCU endpoint as of this writing. It will
later be enhanced so that multiple instances of a tunneling daemon can cater to
different MCUs.

- Enable `COFIG_TUN` and `CONFIG_MCTP` in the kernel KConfig.
- Enable the in-kernel MCTP userspace tools from this [distro feature](https://github.com/openbmc/openbmc/blob/master/meta-phosphor/conf/distro/include/mctp.inc).
- Turn on `enable-mctp-tun` in libmctp to enable the tunneling daemon (mctp-tun)
- mctp-tun can be run as follows
  `mctp-tun vendor_id=0x0955 product_id=0xCF10 class_id=0x0 -v`:
  - `vendor_id` is the Vendor ID in the remote endpoint's USB device descriptor
  - `product_id` is the Product ID in the remote endpoint's USB device
     descriptor
  - `class_id` is currently unused
  - `-v` is an optional argument that can be used to generate verbose Tx/Rx logs
- Set up the in-kernel MCTP stack to route packets to the tunneling daemon (this
  could be done within a systemd service that launches the control daemon with
  an ExecStartPre, for example):

    ``` txt
    #update local endpoint, ex EID 8
    mctp addr add 8 dev tun0
    #update remote endpoint id ex EID 12
    mctp route add 12 via tun0
    #also route and downstream endpoints
    mctp route add 13 via tun0
    #bring up the link for tun0
    mctp link set tun0 up
    ```

- Launch the MCTP control service to perform MCTP discovery
  `/usr/bin/mctp-ctrl -m 1 -t 3 -i 8 -p 12 -x 13 -d 1 -v 1`
- Note that the laucnhing of these services can be done via installing systemd
  service files that launch the MCTP tunneling daemon and the MCTP control
  daemon. Here is an example of both systemd service files.

  MCTP tunnel service (mctp-usb-tun.service):

  ``` ini
  [Unit]

  Description=MCTP USB tunneling daemon

  [Service]
  Restart=always
  Environment=TUN_USB_BINDING_OPTS=null
  EnvironmentFile=-/usr/share/mctp/mctp
  ExecStart=/usr/bin/mctp-tun $TUN_USB_BINDING_OPTS
  SyslogIdentifier=mctp-usb-tun

  [Unit]
  WantedBy=multi-user.target
  ```

  MCTP control service (mctp-usb-ctrl.service):

  ``` ini
  [Unit]
  Description=MCTP USB control daemon

  [Service]
  Restart=always
  Environment=MCTP_USB_CTRL_OPTS=null
  EnvironmentFile=-/usr/share/mctp/mctp
  ExecStart=/usr/bin/mctp-usb-ctrl $MCTP_USB_CTRL_OPTS
  SyslogIdentifier=mctp-usb-ctrl

  [Install]
  WantedBy=multi-user.target

  [Unit]
  Requires=mctp-usb-tun.service
  ```

  Sample environment file (/usr/share/mctp/mctp):

  ``` ini
  # cat /usr/share/mctp/mctp
  TUN_USB_BINDING_OPTS= vendor_id=0x0955 product_id=0xCF10 class_id=0x0
  MCTP_USB_CTRL_OPTS= -m 1 -t 3 -i 8 -p 12 -x 13 -d 1 -v 1
  ```

#### Step3: Check expected D-Bus tree

This assumes we have used the NVBMC MCTP control daemon. EID 12 is the EID for
the MCU bridge and 13 is the EID assigned to the GPU.

``` txt
#  busctl tree xyz.openbmc_project.MCTP.Control.USB -l
`- /xyz
  `- /xyz/openbmc_project
    `- /xyz/openbmc_project/mctp
      |- /xyz/openbmc_project/mctp/0
      | |- /xyz/openbmc_project/mctp/0/12
      | |- /xyz/openbmc_project/mctp/0/13
      `- /xyz/openbmc_project/mctp/USB
```

``` txt
#  busctl introspect xyz.openbmc_project.MCTP.Control.USB /xyz/openbmc_project/mctp/0/13 -l
NAME                                  TYPE      SIGNATURE RESULT/VALUE                                        FLAGS
org.freedesktop.DBus.Introspectable   interface -         -                                                   -
.Introspect                           method    -         s                                                   -
org.freedesktop.DBus.Peer             interface -         -                                                   -
.GetMachineId                         method    -         s                                                   -
.Ping                                 method    -         -                                                   -
org.freedesktop.DBus.Properties       interface -         -                                                   -
.Get                                  method    ss        v                                                   -
.GetAll                               method    s         a{sv}                                               -
.Set                                  method    ssv       -                                                   -
.PropertiesChanged                    signal    sa{sv}as  -                                                   -
xyz.openbmc_project.Common.UUID       interface -         -                                                   -
.UUID                                 property  s         "f72d6fb0-5675-11ed-9b6a-0242ac120002"              const
xyz.openbmc_project.Common.UnixSocket interface -         -                                                   -
.Address                              property  ay        13 0 109 99 116 112 45 117 115 98 45 109 117 120    const
.Protocol                             property  u         0                                                   const
.Type                                 property  u         5                                                   const
xyz.openbmc_project.MCTP.Binding      interface -         -                                                   -
.BindingType                          property  s         "xyz.openbmc_project.MCTP.Binding.BindingTypes.USB" const
xyz.openbmc_project.MCTP.Endpoint     interface -         -                                                   -
.EID                                  property  u         13                                                  const
.MediumType                           property  s         "xyz.openbmc_project.MCTP.Endpoint.MediaTypes.I3C"  const
.NetworkId                            property  u         0                                                   const
.SupportedMessageTypes                property  ay        5 1 5 6 126 127                                     const
xyz.openbmc_project.Object.Enable     interface -         -                                                   -
.Enabled                              property  b         true                                                emits-change writable
```

##### D-Bus interfaces implemented

Each MCTP endpoint D-Bus object implements the following D-Bus interfaces. All
of the interfaces and the respective properties are required by MCTP control
daemon's clients in order to discover and utilize the MCTP endpoint.

- `xyz.openbmc_project.Common.UUID`
  - Property `UUID`, which is the MCTP UUID for the corresponding MCTP endpoint.
- `xyz.openbmc_project.MCTP.Binding`
  - Property `BindingType`, which denotes the binding used by the BMC. For
    example, when the MCTP control daemon is running for the USB binding, this
    shall be `xyz.openbmc_project.MCTP.Binding.BindingTypes.USB`.
- `xyz.openbmc_project.MCTP.Endpoint`. Properties on this interface are as
  reported by the MCU bridge in its routing table.
  - Property `EID` - the MCTP EID for the endpoint.
  - Property `MediumType` - The physical medium type between the MCU and the
    endpoint.
  - Property `NetworkId` - The MCTP network ID, this is always 0 when using the
    NVIDIA supplied MCTP control daemon.
  - Property `SupportedMessageTypes` - An array of supported MCTP message types.
- `xyz.openbmc_project.Object.Enable`
  - Property `Enabled` - To indicate whether the endpoint is in an enabled
    state. If a control daemon implementation does not support dynamically
    disabling endpoints (for ex, via an MCTP discovery notify message), then
    this should always be set to `true`.
- `xyz.openbmc_project.Common.UnixSocket`. This interface is only used when
  using the userspace libmctp demux daemon. This is not needed if using the
  in-kernel MCTP with a tunneling daemon.

In addition to the endpoint D-Bus objects, the control daemon also hosts the
object `/xyz/openbmc_project/mctp/USB` which is as below:

``` txt
# busctl introspect xyz.openbmc_project.MCTP.Control.USB /xyz/openbmc_project/mctp/USB -l
NAME                                   TYPE      SIGNATURE  RESULT/VALUE                                               FLAGS
org.freedesktop.DBus.Introspectable    interface -          -                                                          -
.Introspect                            method    -          s                                                          -
org.freedesktop.DBus.ObjectManager     interface -          -                                                          -
.GetManagedObjects                     method    -          a{oa{sa{sv}}}                                              -
.InterfacesAdded                       signal    oa{sa{sv}} -                                                          -
.InterfacesRemoved                     signal    oas        -                                                          -
org.freedesktop.DBus.Peer              interface -          -                                                          -
.GetMachineId                          method    -          s                                                          -
.Ping                                  method    -          -                                                          -
org.freedesktop.DBus.Properties        interface -          -                                                          -
.Get                                   method    ss         v                                                          -
.GetAll                                method    s          a{sv}                                                      -
.Set                                   method    ssv        -                                                          -
.PropertiesChanged                     signal    sa{sv}as   -                                                          -
xyz.openbmc_project.State.ServiceReady interface -          -                                                          -
.ServiceType                           property  s          "xyz.openbmc_project.State.ServiceReady.ServiceTypes.MCTP" const
.State                                 property  s          "xyz.openbmc_project.State.ServiceReady.States.Enabled"    emits-change writable
```

The interface to note here is `xyz.openbmc_project.State.ServiceReady` which
needs the below properties:

- `ServiceType` is always
  `xyz.openbmc_project.State.ServiceReady.ServiceTypes.MCTP`
- `State` starts with
  `xyz.openbmc_project.State.ServiceReady.States.Starting` and changes to
  `xyz.openbmc_project.State.ServiceReady.States.Enabled` once the control
  daemon has finished discovery.

#### References

- [MCTP design docs](https://github.com/NVIDIA/nvbmc-docs/tree/develop/nvidia/MCTP)

### Telemetry
[NvBMC nsmd](https://github.com/NVIDIA/nsmd) can discover NSM endpoints and
fetch telemetry from them.

#### Step1: Write bitbake recipe

Below is the nsmd reciepe file.
To enable and build the NSMD recipe in your Yocto project, follow these steps:
- Add the Recipe to Your Layer: e.g. meta-nvidia/recipes-nvidia/nsmd/nsmd_git.bb
- Include the Layer in Your Build: Make sure your Yocto project includes the layer containing the NSMD recipe. Edit your bblayers.conf file to include the path to the layer
- Add the Recipe to Your Image: To include the NSMD package in your image, you need to add it to the image recipe. For e.g. use OBMC_IMAGE_EXTRA_INSTALL:append = nsmd in custom image reciepe eg. meta-nvidia/meta-hgxb/recipes-phosphor/images/obmc-phosphor-image.bbappend for platform umbriel.

```
SUMMARY = "Nvidia System Management Daemon"
DESCRIPTION = "Nvidia System Management Daemon"

PR = "r1"
PV = "0.1+git${SRCPV}"

LICENSE = "CLOSED"

inherit meson pkgconfig obmc-phosphor-systemd

DEPENDS += "function2"
DEPENDS += "systemd"
DEPENDS += "sdbusplus"
DEPENDS += "sdeventplus"
DEPENDS += "phosphor-dbus-interfaces"
DEPENDS += "phosphor-logging"
DEPENDS += "nlohmann-json"
DEPENDS += "cli11"
DEPENDS += "libmctp"
DEPENDS += "nvidia-tal"

#EXTRA_OEMESON = "-Dtests=disabled"

SRC_URI = "git://git@gitlab-master.nvidia.com:12051/dgx/bmc/nsmd.git;protocol=ssh;branch=develop"
SRCREV = "5bfa384101d9dc05867779a99b398c602067a9b2"
S = "${WORKDIR}/git"

SYSTEMD_SERVICE:${PN} = "nsmd.service"
FILES:${PN}:append = " ${datadir}/libnsm/instance-db/default"
```

All the above dependencies are mandatory except nvidia-tal.
For nv-tal reference : https://gitlab-master.nvidia.com/dgx/bmc/nvidia-tal
For nv-shmem reference : https://gitlab-master.nvidia.com/dgx/bmc/nv-shmem

##### Meson Build Options in nsmd

| **Option**                           | **Type**   | **Default Value** | **Description**                                                                                                                  |
|--------------------------------------|------------|-------------------|----------------------------------------------------------------------------------------------------------------------------------|
| `tests`                              | `feature`  | `enabled`         | Controls whether the tests are built for the project.                                                                             |
| `stanbyToDC`                         | `feature`  | `enabled`         | Enables features related to transitioning from standby to DC (Direct Current) mode.                                               |
| `shmem`                              | `feature`  | `enabled`         | Enables support for NVIDIA's Shared-Memory Inter-Process Communication (IPC).                                                     |
| `sensor-polling-time`                | `integer`  | `249`             | Specifies the interval time of sensor polling in milliseconds.                                                                    |
| `gpio-name`                          | `string`   | `GPU_BASE_PWR_GD` | Specifies the GPIO name for NSM (Nvidia System Management) operations.                                                            |
| `sensor-polling-time-long-running`   | `integer`  | `1499`            | Specifies the interval time of sensor polling for NSM long-running requests in milliseconds.                                      |
| `mockup-responder`                   | `feature`  | `enabled`         | Enables a mockup responder for testing or simulation purposes.                                                                    |
| `instance-id-expiration-interval`    | `integer`  | `5`               | Specifies how often NSM instance IDs expire in seconds.                                                                           |
| `instance-id-expiration-interval-long-running` | `integer` | `10`        | Specifies how often NSM instance IDs expire for long-running requests in seconds.                                                 |
| `number-of-request-retries`          | `integer`  | `2`               | Specifies the number of retries for NSM requests before failing.                                                                  |
| `response-time-out`                  | `integer`  | `2000`            | Specifies the timeout interval for NSM responses in milliseconds.                                                                 |
| `response-time-out-long-running`     | `integer`  | `3000`            | Specifies the timeout interval for NSM responses during long-running requests in milliseconds.                                    |
| `local-eid`                          | `integer`  | `9`               | Specifies the default local EID (Endpoint Identifier) for local communication.                                                    |
| `aer-error-status-priority`          | `boolean`  | `false`           | Specifies whether to prioritize Aer Error Status Sensors.                                                                         |
| `per-lan-error-count-priority`       | `boolean`  | `false`           | Specifies whether to prioritize Per Lane Error Count Sensors.                                                                     |
| `error-injection-priority`           | `boolean`  | `false`           | Specifies whether to prioritize Error Injection during error testing or handling.                                                 |


Bitbake changes to include all configurations in entity manager

```
FILESEXTRAPATHS:append := "${THISDIR}/files:"

DEPENDS += "python3-pandas-native"
DEPENDS += "python3-openpyxl-native"
DEPENDS += "python3-xlrd-native"

SRC_URI = "git://git@gitlab-master.nvidia.com:12051/dgx/bmc/entity-manager.git;protocol=ssh;branch=develop file://blocklist.json"
SRCREV = "1cfa102bdb0435ade263b3805a2cf3457a054f10"

SRC_URI:append = "
                   file://hgxb_gpu_chassis.json \
                   file://hgxb_static_inventory.json \
                   file://hgxb_instance_mapping.json \
                 "

RDEPENDS:${PN} = " \
        fru-device \
        "

do_install:append() {
     # Remove unnecessary config files. EntityManager spends significant time parsing these.
     rm -f ${D}/usr/share/entity-manager/configurations/*.json

     install -m 0444 ${WORKDIR}/hgxb_gpu_chassis.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_static_inventory.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_instance_mapping.json ${D}/usr/share/entity-manager/configurations
}

```

#### Step2: Write Configuration Files

##### STATIC INVENTORY AND DYNAMIC INVENTORY
NSM supports static inventory creation. Static inventory means that inventory objects will always be populated to D-Bus no matter if the communication of the NSM Device is ready or not.

The properties of PDIs will be initialized to default value and they should  be updated to correct value once the device shows up on MCTP network(e.g. power on).

What inventory object should be created by nsmd can be configurable by EM json file and whether inventory is static or dynamic is also configured through EM json file or by “probe” property of EM json with more specific.

For static Inventory, the EM json probe rule should not be the condition depending on if EID enumerated or not. And the UUID property of every config PDIs should be in the format, “DEVICE_TYPE=X:INSTANCE_ID=Y” for nsmd to match the config PDI to correct nsmDevice when the device is enumerated.

For dynamic inventory, the EM json probe rule should be "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': X})" for the condition when nsmd created FruDevice PDI for the enumerated EID. And the UUID property of every config PDIs should be “$UUID”.

Static inventory configuration : https://gitlab-master.nvidia.com/dgx/bmc/openbmc/-/blob/develop/meta-nvidia/meta-hgxb/recipes-phosphor/configuration/entity-manager/files/hgxb_static_inventory.json

e.g.

```
{
    "Exposes": [],
    "Name": "NSM_DEV_GPU_0",
    "Probe": <Match something from the B40 FRU EEPROM>,
    "Type": "NSM_Configs",
    "xyz.openbmc_project.NsmDevice": {
        "DEVICE_TYPE": 0,
        "INSTANCE_NUMBER": 0,
        "UUID": "STATIC:0:0"
}

```

DEVICE_TYPE is unique value which identifies whether it is gpu, fpga, qm3 etc
INSTANCE_NUMBER is unique number to identify different instances of same device type
For Static inventory as discussed above UUID is is format DEVICE_TYPE=X:INSTANCE_ID=Y , so for device type 0 and instance 3 it will be "UUID": "STATIC:0:3"

dynamic inventory : https://gitlab-master.nvidia.com/dgx/bmc/openbmc/-/blob/develop/meta-nvidia/meta-hgxb/recipes-phosphor/configuration/entity-manager/files/hgxb_gpu_chassis.json

```
{
    "Exposes": [
      {
        "Name": "HGX_GPU_SXM_$INSTANCE_NUMBER + 1",
        "Type": "NSM_Chassis",
        "UUID": "$UUID",
        "DeviceType": "$DEVICE_TYPE",
        "Dimension": {
          "Type": "NSM_Dimension"
        },
        "Location": {
          "Type": "NSM_Location",
          "LocationType": "xyz.openbmc_project.Inventory.Decorator.Location.LocationTypes.Embedded"
        },
        "Asset": {
          "Type": "NSM_Asset",
          "Manufacturer": "NVIDIA"
        }
      }
    ],
    "Probe": "map something to platform FRU EEPROM",
    "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1",
    "Type": "chassis",
    "Parent_Chassis": "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
    "xyz.openbmc_project.Inventory.Decorator.Instance": {
      "InstanceNumber": "$INSTANCE_NUMBER"
    }
  }

```

Here we are creating configuration pdi related to gpu chassis to be consumed by nsmd. we create sensor of type NSM_Dimension, NSM_Location, NSM_Asset etc here. we pass on property values like Manufacturer, LocationType with it.

##### GPU INDEX MAPPING CONFIG FILE

https://gitlab-master.nvidia.com/dgx/bmc/openbmc/-/blob/develop/meta-nvidia/meta-hgxb/recipes-phosphor/configuration/entity-manager/files/hgxb_instance_mapping.json

Adding support to have ability to update device instanceID via EM json configuration based on either of below mentioned fields.
1. Instance ID [received from queryDeviceIdentification cmd]
2. EID
3. UUID

NOTE: Priority is given in above order itself.

In below examples:
```
Eid: 30 --> Mocks GPU with instanceId 1 [as per mapping instanceID 1 should have 5 as instanceID]
Eid: 31 --> Mocks GPU with instanceId 4 [as per mapping instanceID 4 should have 0 as instanceID]
```

```
    Sample EM json for 8 gpu:
[
  {
    "Exposes": [
      {
        "Name": "GPUMapping",
        "Type": "NSM_GetInstanceIDByDeviceInstanceID",
        "MappingArr": [
          4,
          5,
          6,
          7,
          0,
          1,
          2,
          3
        ]
      },


      OR


      {
        "Name": "GPUMapping",
        "Type": "NSM_GetInstanceIDByDeviceEID",
        "MappingArr": [
          30,
          31,
          28,
          29,
          32,
          33,
          35,
          34
        ]
      }



      OR


      {
        "Name": "GPUMapping",
        "Type": "NSM_GetInstanceIDByDeviceUUID",
        "MappingArr": [
          "24000000-0000-0000-0000-000000000000",
          ....,
          ....,
          .
          .
          .
          .
          "24000000-0000-0000-0000-0000000023423"
        ]
      }
    ],
    "Name": "Mapping",
    "Probe": "TRUE",
    "Type": "NSM_Configs"
  }
]

```

Expected result on dbus

```
root@hgxb:~# busctl tree xyz.openbmc_project.EntityManager
`- /xyz
  `- /xyz/openbmc_project
    |- /xyz/openbmc_project/EntityManager
    `- /xyz/openbmc_project/inventory
      `- /xyz/openbmc_project/inventory/system
        |- /xyz/openbmc_project/inventory/system/chassis
........................................
        `- /xyz/openbmc_project/inventory/system/nsm_configs
          |- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping
          | |- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/GPUMapping
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_0

**OPTION 1**
root@hgxb:~# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/GPUMapping
NAME                                                    TYPE      SIGNATURE RESULT/VALUE            FLAGS
org.freedesktop.DBus.Introspectable                     interface -         -                       -
.Introspect                                             method    -         s                       -
org.freedesktop.DBus.Peer                               interface -         -                       -
.GetMachineId                                           method    -         s                       -
.Ping                                                   method    -         -                       -
org.freedesktop.DBus.Properties                         interface -         -                       -
.Get                                                    method    ss        v                       -
.GetAll                                                 method    s         a{sv}                   -
.Set                                                    method    ssv       -                       -
.PropertiesChanged                                      signal    sa{sv}as  -                       -
xyz.openbmc_project.Configuration.NSM_GetInstanceIDByD  interface -         -                       -
.MappingArr                                             property  at        8 4 5 6 7 0 1 2 3       emits-change
.Name                                                   property  s         "GPUMapping"            emits-change
.Type                                                   property  s         "NSM_GetInstanceIDByDev emits-change


**OPTION 2**
root@hgxb:~# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/GPUMapping
NAME                                              TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.freedesktop.DBus.Introspectable               interface -         -                                        -
.Introspect                                       method    -         s                                        -
org.freedesktop.DBus.Peer                         interface -         -                                        -
.GetMachineId                                     method    -         s                                        -
.Ping                                             method    -         -                                        -
org.freedesktop.DBus.Properties                   interface -         -                                        -
.Get                                              method    ss        v                                        -
.GetAll                                           method    s         a{sv}                                    -
.Set                                              method    ssv       -                                        -
.PropertiesChanged                                signal    sa{sv}as  -                                        -
xyz.openbmc_project.Configuration.NSM_UUIDMapping interface -         -                                        -
.MappingArr                                       property  as        1 "24000000-0000-0000-0000-000000000000" emits-change
.Name                                             property  s         "GPUMapping"                             emits-change
.Type                                             property  s         "NSM_UUIDMapping"                        emits-change


**OPTION 3**
root@hgxb:~# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/GPUMapping
NAME                                             TYPE      SIGNATURE         RESULT/VALUE                  FLAGS
org.freedesktop.DBus.Introspectable              interface -                 -                               -
.Introspect                                      method    -                 s                               -
org.freedesktop.DBus.Peer                        interface -                 -                               -
.GetMachineId                                    method    -                 s                               -
.Ping                                            method    -                 -                               -
org.freedesktop.DBus.Properties                  interface -                 -                               -
.Get                                             method    ss                v                               -
.GetAll                                          method    s                 a{sv}                           -
.Set                                             method    ssv               -                               -
.PropertiesChanged                               signal    sa{sv}as          -                               -
xyz.openbmc_project.Configuration.NSM_GetInstan  interface -                 -                               -
.MappingArr                                      property  at                2 30 31 28 29 32 35 34          emits-change
.Name                                            property  s                 "GPUMapping"                    emits-change
.Type                                            property  s                 "NSM_GetInstance                emits-change
```


#### Step3: Check expected D-Bus tree (with all config written)

```
root@hgxb:/usr/share/entity-manager/configurations# busctl tree xyz.openbmc_project.EntityManager
`- /xyz
  `- /xyz/openbmc_project
    |- /xyz/openbmc_project/EntityManager
    `- /xyz/openbmc_project/inventory
      `- /xyz/openbmc_project/inventory/system
        |- /xyz/openbmc_project/inventory/system/chassis
        | `- /xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM_1
        |   `- /xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM_1/HGX_GPU_SXM_1
        `- /xyz/openbmc_project/inventory/system/nsm_configs
          |- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping
        | |- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/GPUMapping
          `- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_0
```


#### FULL BLACKWELL GPU CONFIG

```
[
  {
    "Exposes": [
      {
        "Name": "HGX_GPU_SXM_$INSTANCE_NUMBER + 1",
        "Type": "NSM_Chassis",
        "UUID": "$UUID",
        "DeviceType": "$DEVICE_TYPE",
        "Chassis": {
          "Type": "NSM_Chassis",
          "DEVICE_UUID": "$DEVICE_UUID"
        },
        "Asset": {
          "Type": "NSM_Asset",
          "Manufacturer": "NVIDIA"
        },
        "Dimension": {
          "Type": "NSM_Dimension"
        },
        "Location": {
          "Type": "NSM_Location",
          "LocationType": "xyz.openbmc_project.Inventory.Decorator.Location.LocationTypes.Embedded"
        },
        "LocationCode": {
          "Type": "NSM_LocationCode",
          "LocationCode": "SXM$INSTANCE_NUMBER + 1"
        },
        "ChassisType": {
          "Type": "NSM_ChassisType",
          "ChassisType": "xyz.openbmc_project.Inventory.Item.Chassis.ChassisType.Module"
        },
        "Health": {
          "Type": "NSM_Health",
          "Health": "xyz.openbmc_project.State.Decorator.Health.HealthType.OK"
        },
        "PowerLimit": {
          "Type": "NSM_PowerLimit",
          "Priority": false
        },
        "PrettyName": {
          "Type": "NSM_PrettyName",
          "Name": "GPU_SXM_$INSTANCE_NUMBER + 1"
        }
      },
      {
        "ChassisName": "HGX_GPU_SXM_$INSTANCE_NUMBER + 1",
        "Name": "Assembly0",
        "Type": "NSM_ChassisAssembly",
        "UUID": "$UUID",
        "Area": {
          "Type": "NSM_Area",
          "PhysicalContext": "xyz.openbmc_project.Inventory.Decorator.Area.PhysicalContextType.GPU"
        },
        "Asset": {
          "Type": "NSM_Asset",
          "Name": "GPU Board Assembly",
          "Vendor": "NVIDIA"
        },
        "Health": {
          "Type": "NSM_Health",
          "Health": "xyz.openbmc_project.State.Decorator.Health.HealthType.OK"
        },
        "Location": {
          "Type": "NSM_Location",
          "LocationType": "xyz.openbmc_project.Inventory.Decorator.Location.LocationTypes.Embedded"
        }
      },
      {
        "ChassisName": "HGX_GPU_SXM_$INSTANCE_NUMBER + 1",
        "Name": "Assembly1",
        "Type": "NSM_ChassisAssembly",
        "UUID": "$UUID",
	"DeviceAssembly": true,
        "Area": {
          "Type": "NSM_Area",
          "PhysicalContext": "xyz.openbmc_project.Inventory.Decorator.Area.PhysicalContextType.GPU"
        },
        "Asset": {
          "Type": "NSM_Asset",
          "Name": "GPU Device Assembly",
          "Vendor": "NVIDIA"
        },
        "Health": {
          "Type": "NSM_Health",
          "Health": "xyz.openbmc_project.State.Decorator.Health.HealthType.OK"
        },
        "Location": {
          "Type": "NSM_Location",
          "LocationType": "xyz.openbmc_project.Inventory.Decorator.Location.LocationTypes.Embedded"
        }
      },
      {
        "ChassisName": "HGX_GPU_SXM_$INSTANCE_NUMBER + 1",
        "Name": "GPU_SXM_$INSTANCE_NUMBER + 1",
        "Type": "NSM_ChassisPCIeDevice",
        "UUID": "$UUID",
        "DEVICE_UUID": "$DEVICE_UUID",
        "Asset": {
          "Type": "NSM_Asset",
          "Name": "HGX_GPU_SXM_$INSTANCE_NUMBER + 1",
          "Manufacturer": "NVIDIA"
        },
        "Associations": [
          {
            "Forward": "chassis",
            "Backward": "pciedevice",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM_$INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "connected_port",
            "Backward": "connected_pciedevice",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_$CONNECTED_RETIMER_INSTANCE_NUM/Switches/PCIeRetimer_$CONNECTED_RETIMER_INSTANCE_NUM/Ports/Down_0"
          }
        ],
        "Health": {
          "Type": "NSM_Health",
          "Health": "xyz.openbmc_project.State.Decorator.Health.HealthType.OK"
        },
        "PCIeDevice": {
          "Type": "NSM_PCIeDevice",
          "DeviceType": "SingleFunction",
          "DeviceIndex": 0,
          "Priority": false,
          "Functions": [
            0
          ]
        },
        "LTSSMState": {
          "Type": "NSM_LTSSMState",
          "DeviceIndex": 0,
          "Priority": false,
          "InventoryObjPath": "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_$CONNECTED_RETIMER_INSTANCE_NUM/Switches/PCIeRetimer_$CONNECTED_RETIMER_INSTANCE_NUM/Ports/Down_0"
        },
        "ClockOutputEnableState": {
          "Type": "NSM_ClockOutputEnableState",
          "InstanceNumber": "$INSTANCE_NUMBER",
          "DeviceType": "$DEVICE_TYPE",
          "Priority": false
        }
      },
      {
        "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1 TEMP_0",
        "Type": "NSM_Temp",
        "Associations": [
          {
            "Forward": "chassis",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "processor",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM $INSTANCE_NUMBER + 1"
          }
        ],
        "UUID": "$UUID",
        "PhysicalContext": "GPU",
        "Aggregated": true,
        "SensorId": 0,
        "Priority": true
      },
      {
        "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1 TEMP_1",
        "Type": "NSM_Temp",
        "Associations": [
          {
            "Forward": "chassis",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "processor",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM $INSTANCE_NUMBER + 1"
          }
        ],
        "ThermalParameters": [
          {
            "Name": "LowerCaution",
            "Dynamic": true,
            "Type": "NSM_ThermalParameter",
            "ParameterId": 4,
            "PeriodicUpdate": false
          },
          {
            "Name": "LowerCritical",
            "Dynamic": true,
            "Type": "NSM_ThermalParameter",
            "ParameterId": 1,
            "PeriodicUpdate": false
          },
          {
            "Name": "LowerFatal",
            "Dynamic": true,
            "Type": "NSM_ThermalParameter",
            "ParameterId": 2,
            "PeriodicUpdate": false
          }
        ],
        "UUID": "$UUID",
        "PhysicalContext": "GPU",
        "Implementation": "Synthesized",
        "ReadingBasis": "Headroom",
        "Description": "Thermal Limit(TLIMIT) Temperature is the distance in deg C from the GPU temperature to the first throttle limit.",
        "Aggregated": true,
        "SensorId": 2,
        "Priority": true
      },
      {
        "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1 DRAM_0_Temp_0",
        "Type": "NSM_Temp",
        "Associations": [
          {
            "Forward": "chassis",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "processor",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "memory",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/memory/GPU_SXM $INSTANCE_NUMBER + 1 DRAM_0"
          }
        ],
        "ThermalParameters": [
          {
            "Name": "UpperCritical",
            "Dynamic": false,
            "Value": 95.0
          }
        ],
        "UUID": "$UUID",
        "PhysicalContext": "GPU",
        "Aggregated": true,
        "SensorId": 1,
        "Priority": true
      },
      {
        "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1 Power_0",
        "Type": "NSM_Power",
        "CompositeNumericSensors": [
          "/xyz/openbmc_project/sensors/power/HGX_Chassis_0_TotalGPU_Power_0"
        ],
        "Associations": [
          {
            "Forward": "chassis",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "processor",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM $INSTANCE_NUMBER + 1"
          }
        ],
        "PeakValue": {
            "SensorId": 0,
            "AveragingInterval": 0,
            "Aggregated": true,
            "Priority": true
        },
        "UUID": "$UUID",
        "PhysicalContext": "GPU",
        "MaxAllowableOperatingValue": 1020.0,
        "Aggregated": true,
        "SensorId": 0,
        "AveragingInterval": 0,
        "Priority": true
      },
      {
        "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1 DRAM_0_Power_0",
        "Type": "NSM_Power",
        "Associations": [
          {
            "Forward": "chassis",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "processor",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "memory",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/memory/GPU_SXM $INSTANCE_NUMBER + 1 DRAM_0"
          }
        ],
        "UUID": "$UUID",
        "PhysicalContext": "GPU",
        "Aggregated": true,
        "SensorId": 1,
        "AveragingInterval": 0,
        "Priority": true
      },
      {
        "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1 Energy_0",
        "Type": "NSM_Energy",
        "Associations": [
          {
            "Forward": "chassis",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "processor",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM $INSTANCE_NUMBER + 1"
          }
        ],
        "UUID": "$UUID",
        "PhysicalContext": "GPU",
        "Aggregated": true,
        "SensorId": 0,
        "Priority": true
      },
      {
        "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1 Voltage_0",
        "Type": "NSM_Voltage",
        "Associations": [
          {
            "Forward": "chassis",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "processor",
            "Backward": "all_sensors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM $INSTANCE_NUMBER + 1"
          }
        ],
        "UUID": "$UUID",
        "PhysicalContext": "GPU",
        "Aggregated": true,
        "SensorId": 0,
        "Priority": false
      },
      {
        "Name": "HGX_Driver_GPU_SXM_$INSTANCE_NUMBER + 1",
        "Type": "NSM_GPU_SWInventory",
        "Associations": [
          {
            "Forward": "inventory",
            "Backward": "software",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "software_version",
            "Backward": "updateable",
            "AbsolutePath": "/xyz/openbmc_project/software"
          }
        ],
        "UUID": "$UUID",
        "Manufacturer": "Nvidia"
      },
      {
        "Name": "GlobalEventSetting",
        "Type": "NSM_EventSetting",
        "UUID": "$UUID",
        "EventGenerationSetting": 2
      },
      {
        "Name": "deviceCapabilityDiscoveryEventSetting",
        "Type": "NSM_EventConfig",
        "MessageType": 0,
        "UUID": "$UUID",
        "SubscribedEventIDs": [
          1
        ],
        "AcknowledgementEventIds": []
      },
      {
        "Name": "PlatformEnvironmentEventSetting",
        "Type": "NSM_EventConfig",
        "MessageType": 3,
        "UUID": "$UUID",
        "SubscribedEventIDs": [
          0,
          1
        ]
      },
      {
        "Name": "XIDEventSetting",
        "Type": "NSM_Event_XID",
        "UUID": "$UUID",
        "OriginOfCondition": "/redfish/v1/Chassis/HGX_GPU_SXM_$INSTANCE_NUMBER + 1",
        "MessageId": "ResourceEvent.1.0.ResourceErrorsDetected",
        "Severity": "Critical",
        "LoggingNamespace": "GPU_SXM $INSTANCE_NUMBER + 1 XID",
        "Resolution": "Regarding XID documentation and further actions please refer to XID and sXID Catalog for NVIDIA Data Center Products (NVOnline: 1115699)",
        "MessageArgs": [
          "GPU_SXM_$INSTANCE_NUMBER + 1 Driver Event Message",
          "[{Timestamp}][{SequenceNumber}][{Flags:x}] XID {EventMessageReason} {MessageTextString}"
        ]
      },
      {
        "Name": "ResetRequiredEventSetting",
        "Type": "NSM_Event_Reset_Required",
        "UUID": "$UUID",
        "OriginOfCondition": "/redfish/v1/Chassis/HGX_GPU_SXM_$INSTANCE_NUMBER + 1",
        "MessageId": "Base.1.13.ResetRequired",
        "Severity": "Critical",
        "LoggingNamespace": "GPU_SXM_$INSTANCE_NUMBER + 1",
        "Resolution": "Reset the GPU or power cycle the Baseboard.",
        "MessageArgs": [
          "GPU_SXM_$INSTANCE_NUMBER + 1",
          "ForceRestart"
        ]
      },
      {
        "Name": "NVLink",
        "Type": "NSM_NVLink",
        "ParentObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
        "DeviceType": "$DEVICE_TYPE",
        "UUID": "$UUID",
        "Priority": false,
        "Count": 18
      },
      {
        "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1 ClockLimit_0",
        "Type": "NSM_ControlClockLimit_0",
        "PhysicalContext": "xyz.openbmc_project.Inventory.Decorator.Area.PhysicalContextType.GPU",
        "InventoryObjPath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM_$INSTANCE_NUMBER + 1",
        "Associations": [
          {
            "Forward": "parent_chassis",
            "Backward": "clock_controls",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1"
          }
        ],
        "UUID": "$UUID",
        "ClockMode": "com.nvidia.ClockMode.Mode.MaximumPerformance",
        "Priority": false
      },
      {
        "Name": "GPU_$INSTANCE_NUMBER + 1 Processor",
        "Type": "NSM_Processor",
        "Associations": [
          {
            "Forward": "parent_chassis",
            "Backward": "all_processors",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "system_interface",
            "Backward": "",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM $INSTANCE_NUMBER + 1 /PCIeDevices/GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "all_memory",
            "Backward": "parent_processor",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/memory/HGX_GPU_SXM $INSTANCE_NUMBER + 1 DRAM_0"
          },
          {
            "Forward": "all_switches",
            "Backward": "processor",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/fabrics/HGX_NVLinkFabric_0/Switches/NVSwitch_0"
          },
          {
            "Forward": "all_switches",
            "Backward": "processor",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/fabrics/HGX_NVLinkFabric_0/Switches/NVSwitch_1"
          }
        ],
        "UUID": "$UUID",
        "DEVICE_UUID": "$DEVICE_UUID",
        "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
        "Asset": {
          "Type": "NSM_Asset",
          "Manufacturer": "NVIDIA"
        },
        "Location": {
          "Type": "NSM_Location",
          "LocationType": "xyz.openbmc_project.Inventory.Decorator.Location.LocationTypes.Embedded"
        },
        "LocationCode": {
          "Type": "NSM_LocationCode",
          "LocationCode": "SXM$INSTANCE_NUMBER + 1"
        },
        "MIGMode": {
          "Type": "NSM_MIG",
          "Priority": false
        },
        "PortDisableFuture": {
          "Type": "NSM_PortDisableFuture",
          "Priority": false
        },
        "ECCMode": {
          "Type": "NSM_ECC",
          "Priority": false
        },
        "PowerCap": {
          "Type": "NSM_PowerCap",
          "Priority": false,
          "CompositeNumericSensors": [
            "/xyz/openbmc_project/inventory/system/chassis/power/control/TotalGPU_Power_0"
          ]
        },
	"PowerSmoothing": {
          "Type": "NSM_PowerSmoothing",
          "Priority": false
        },
        "PCIe": {
          "Type": "NSM_PCIe",
          "Priority": false,
          "DeviceId": 0,
          "Count": 1
        },
        "CpuOperatingConfig": {
          "Type": "NSM_CpuOperatingConfig",
          "Priority": false
        },
        "ProcessorPerformance": {
          "Type": "NSM_ProcessorPerformance",
          "Priority": false,
          "DeviceId": 0
        },
        "MemCapacityUtil": {
          "Type": "NSM_MemCapacityUtil",
          "Priority": false
        },
        "InbandReconfigPermissions": {
          "Type": "NSM_InbandReconfigPermissions",
          "Priority": false,
          "Features": [
            "InSystemTest",
            "FusingMode",
            "CCMode",
            "BAR0Firewall",
            "CCDevMode",
            "TGPCurrentLimit",
            "TGPRatedLimit",
            "TGPMaxLimit",
            "TGPMinLimit",
            "ClockLimit",
            "NVLinkDisable",
            "ECCEnable",
            "PCIeVFConfiguration",
            "RowRemappingAllowed",
            "RowRemappingFeature",
            "HBMFrequencyChange",
            "HULKLicenseUpdate",
            "ForceTestCoupling",
            "BAR0TypeConfig",
            "EDPpScalingFactor",
            "PowerSmoothingPrivilegeLevel1",
            "PowerSmoothingPrivilegeLevel2"
           ]
        },
        "TotalNvLinksCount": {
          "Type": "NSM_TotalNvLinksCount",
          "Priority": false
        }
      },
      {
        "Name": "GPU_$INSTANCE_NUMBER + 1 Memory",
        "Type": "NSM_Memory",
        "Associations": [
          {
            "Forward": "parent_processor",
            "Backward": "all_memory",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM $INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "parent_chassis",
            "Backward": "all_memory",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/chassis/HGX_GPU_SXM_$INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "all_sensors",
            "Backward": "memory",
            "AbsolutePath": "/xyz/openbmc_project/sensors/temperature/HGX_GPU_SXM_$INSTANCE_NUMBER + 1_DRAM_0_Temp_0"
          },
          {
            "Forward": "all_sensors",
            "Backward": "memory",
            "AbsolutePath": "/xyz/openbmc_project/sensors/power/HGX_GPU_SXM_$INSTANCE_NUMBER + 1_DRAM_0_Power_0"
          }
        ],
        "ErrorCorrection": "xyz.openbmc_project.Inventory.Item.Dimm.Ecc.SingleBitECC",
        "DeviceType": "xyz.openbmc_project.Inventory.Item.Dimm.DeviceType.HBM",
        "UUID": "$UUID",
        "Priority": false,
        "InventoryObjPath": "/xyz/openbmc_project/inventory/system/memory/GPU_SXM_$INSTANCE_NUMBER + 1",
        "RowRemapping": {
          "Type": "NSM_RowRemapping",
          "Priority": false
        },
        "ECCMode": {
          "Type": "NSM_ECC",
          "Priority": false
        },
        "MemCapacityUtil": {
          "Type": "NSM_MemCapacityUtil",
          "Priority": false
        }
      },
      {
        "Name": "GPU_$INSTANCE_NUMBER + 1 PCIe_0",
        "Type": "NSM_GPU_PCIe_0",
        "UUID": "$UUID",
        "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
        "Health": "xyz.openbmc_project.State.Decorator.Health.HealthType.OK",
        "ChasisPowerState": "xyz.openbmc_project.State.Chassis.PowerState.On",
        "DeviceIndex": 0,
        "ClearableScalarGroup": [
          2,
          3,
          4
        ],
        "Associations": [
          {
            "Forward": "parent_device",
            "Backward": "all_states",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1"
          },
          {
            "Forward": "associated_switch",
            "Backward": "connected_port",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_$CONNECTED_RETIMER_INSTANCE_NUM/Switches/PCIeRetimer_$CONNECTED_RETIMER_INSTANCE_NUM"
          },
          {
            "Forward": "switch_port",
            "Backward": "processor_port",
            "AbsolutePath": "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_$CONNECTED_RETIMER_INSTANCE_NUM/Switches/PCIeRetimer_$CONNECTED_RETIMER_INSTANCE_NUM/Ports/Down_0"
          }
        ],
        "PortInfo": {
          "Type": "NSM_PortInfo",
          "PortType": "xyz.openbmc_project.Inventory.Decorator.PortInfo.PortType.UpstreamPort",
          "PortProtocol": "xyz.openbmc_project.Inventory.Decorator.PortInfo.PortProtocol.PCIe",
          "Priority": false
        }
      },
      {
        "Name": "GPU_$INSTANCE_NUMBER + 1 GPM_Aggregated_Metrics",
        "Type": "NSM_GPMMetrics",
        "UUID": "$UUID",
        "Priority": false,
        "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
        "RetrievalSource": 1,
        "GpuInstance": 255,
        "ComputeInstance": 255,
        "MetricsBitfield": [255, 255, 31],
        "MemoryBandwidth": true,
        "MemoryInventoryObjPath": "/xyz/openbmc_project/inventory/system/memory/GPU_SXM $INSTANCE_NUMBER + 1 DRAM_0",
        "PerInstanceMetrics": [
          {
            "Name": "GPU_$INSTANCE_NUMBER + 1 GPM_NVDEC_PerInstance_Metrics",
            "Type": "NSM_GPMPerInstanceMetrics",
            "Priority": false,
            "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
            "RetrievalSource": 2,
            "GpuInstance": 255,
            "ComputeInstance": 255,
            "Metric": "NVDEC",
            "MetricId": 14,
            "InstanceBitfield": 255
          },
          {
            "Name": "GPU_$INSTANCE_NUMBER + 1 GPM_NVJPG_PerInstance_Metrics",
            "Type": "NSM_GPMPerInstanceMetrics",
            "Priority": false,
            "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
            "RetrievalSource": 2,
            "GpuInstance": 255,
            "ComputeInstance": 255,
            "Metric": "NVJPG",
            "MetricId": 15,
            "InstanceBitfield": 255
          }
        ]
      },
      {
        "Name": "GPU_$INSTANCE_NUMBER + 1 GPM_Port_Metrics",
        "Type": "NSM_GPMPortMetrics",
        "UUID": "$UUID",
        "Priority": false,
        "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
        "RetrievalSource": 0,
        "GpuInstance": 255,
        "ComputeInstance": 255,
        "Metrics": [
          "NVLinkRawTxBandwidthGbps",
          "NVLinkDataTxBandwidthGbps",
          "NVLinkRawRxBandwidthGbps",
          "NVLinkDataRxBandwidthGbps"
        ],
        "Ports": [
          0,
          1,
          2,
          3,
          4,
          5,
          6,
          7,
          8,
          9,
          10,
          11,
          12,
          13,
          14,
          15,
          16,
          17
        ],
        "InstanceBitfield": 262143
      }
    ],
    "Probe": "xyz.openbmc_project.NsmDevice({'DEVICE_TYPE': 0})",
    "Name": "HGX_GPU_SXM $INSTANCE_NUMBER + 1",
    "Type": "chassis",
    "Parent_Chassis": "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
    "xyz.openbmc_project.Inventory.Decorator.Instance": {
      "InstanceNumber": "$INSTANCE_NUMBER"
    }
  }
]
```


##### D-Bus interfaces implemented
[xyz.openbmc_project.Association.Definitions]()
[xyz.openbmc_project.Common.UUID]()
[xyz.openbmc_project.Configuration.NsmDeviceAssociation]()
[xyz.openbmc_project.Control.Mode]()
[xyz.openbmc_project.Control.Power.Cap]()
[xyz.openbmc_project.Control.Power.Mode]()
[xyz.openbmc_project.Control.Processor.Reset]()
[xyz.openbmc_project.FruDevice]()
[xyz.openbmc_project.Inventory.Decorator.Area]()
[xyz.openbmc_project.Inventory.Decorator.Asset]()
[xyz.openbmc_project.Inventory.Decorator.Dimension]()
[xyz.openbmc_project.Inventory.Decorator.FpgaType]()
[xyz.openbmc_project.Inventory.Decorator.Location]()
[xyz.openbmc_project.Inventory.Decorator.LocationCode]()
[xyz.openbmc_project.Inventory.Decorator.PCIeRefClock]()
[xyz.openbmc_project.Inventory.Decorator.PortInfo]()
[xyz.openbmc_project.Inventory.Decorator.PortState]()
[xyz.openbmc_project.Inventory.Decorator.PortWidth]()
[xyz.openbmc_project.Inventory.Decorator.PowerLimit]()
[xyz.openbmc_project.Inventory.Decorator.Revision]()
[xyz.openbmc_project.Inventory.Item]()
[xyz.openbmc_project.Inventory.Item.Accelerator]()
[xyz.openbmc_project.Inventory.Item.Assembly]()
[xyz.openbmc_project.Inventory.Item.Chassis]()
[xyz.openbmc_project.Inventory.Item.Cpu.OperatingConfig]()
[xyz.openbmc_project.Inventory.Item.Dimm]()
[xyz.openbmc_project.Inventory.Item.Dimm.MemoryMetrics]()
[xyz.openbmc_project.Inventory.Item.Endpoint]()
[xyz.openbmc_project.Inventory.Item.Fabric]()
[xyz.openbmc_project.Inventory.Item.NetworkInterface]()
[xyz.openbmc_project.Inventory.Item.PCIeDevice]()
[xyz.openbmc_project.Inventory.Item.PCIeSlot]()
[xyz.openbmc_project.Inventory.Item.PersistentMemory]()
[xyz.openbmc_project.Inventory.Item.Port]()
[xyz.openbmc_project.Inventory.Item.Switch]()
[xyz.openbmc_project.Inventory.Item.Zone]()
[xyz.openbmc_project.MCTP.UUID]()
[xyz.openbmc_project.Memory.MemoryECC]()
[xyz.openbmc_project.Metrics.IBPort]()
[xyz.openbmc_project.Metrics.PortMetricsOem2]()
[xyz.openbmc_project.Metrics.PortMetricsOem3]()
[xyz.openbmc_project.PCIe.LTSSMState]()
[xyz.openbmc_project.PCIe.PCIeECC]()
[xyz.openbmc_project.Sensor.Threshold.Critical]()
[xyz.openbmc_project.Sensor.Threshold.HardShutdown]()
[xyz.openbmc_project.Sensor.Threshold.Warning]()
[xyz.openbmc_project.Sensor.Type]()
[xyz.openbmc_project.Sensor.Value]()
[xyz.openbmc_project.Software.Settings]()
[xyz.openbmc_project.Software.Version]()
[xyz.openbmc_project.State.Chassis]()
[xyz.openbmc_project.State.Decorator.Health]()
[xyz.openbmc_project.State.Decorator.OperationalStatus]()
[xyz.openbmc_project.State.ProcessorPerformance]()
[xyz.openbmc_project.State.ServiceReady]()
[xyz.openbmc_project.Time.EpochTime]()
[com.nvidia.Common.ClearPowerCap]()
[com.nvidia.Edpp]()
[com.nvidia.GPMMetrics]()
[com.nvidia.InbandReconfigSettings]()
[com.nvidia.MemoryRowRemapping]()
[com.nvidia.MigMode]()
[com.nvidia.NVLink.NVLinkMetrics]()
[com.nvidia.NVLink.NVLinkRefClock]()

#### References
- [Bitbake Recipe]()
- [Single GPU configuration]()
