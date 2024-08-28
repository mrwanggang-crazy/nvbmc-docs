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

#### Step2: Write Configuration Files
- Enable `COFIG_TUN` and `CONFIG_MCTP` in the kernel KConfig.
- Enable the in-kernel MCTP userspace tools from this [distro feature](https://github.com/openbmc/openbmc/blob/master/meta-phosphor/conf/distro/include/mctp.inc).
- Turn on `enable-mctp-tun` in libmctp to enable the tunneling daemon (mctp-tun).
- mctp-tun can be run as follows `mctp-tun vendor_id=0x0955 product_id=0xFFFF class_id=0x0 -v`:
    - `vendor_id` is the Vendor ID in the remote endpoint's USB device descriptor.
    - `product_id` is the Product ID in the remote endpoint's USB device descriptor.
    - `class_id` is currently unused.
    - `-v` is an optional argument that can be used to generate verbose Tx/Rx logs.
- Set up the in-kernel MCTP stack to route packets to the tunneling daemon:
    ```
    #update local endpoint, ex EID 8
    mctp addr add 8 dev tun0
    #update remote endpoint id ex EID 12
    mctp route add 12 via tun0
    #bring up the link for tun0
    mctp link set tun0 up
    ```

#### Step3: Check expected D-Bus tree

##### D-Bus interfaces implemented

#### References
- [Bitbake Recipe]()
- [MCTP design docs](https://github.com/NVIDIA/nvbmc-docs/tree/develop/nvidia/MCTP)

### Telemetry
[NvBMC nsmd](https://github.com/NVIDIA/nsmd) can discover NSM endpoints and
fetch telemetry from them.

NSM commands listed in the table below require an entity-manager configuration
to function correctly:
| Type | Command                 | Configuration |
| ---- | ----------------------- | ------------- |
| 3    | Get Temperature Reading | [Link]()      |
|      |                         |               |
|      |                         |               |

#### Step1: Write bitbake recipe

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

bitbake changes to include all configurations in entity manager

```
FILESEXTRAPATHS:append := "${THISDIR}/files:"

DEPENDS += "python3-pandas-native"
DEPENDS += "python3-openpyxl-native"
DEPENDS += "python3-xlrd-native"

SRC_URI = "git://git@gitlab-master.nvidia.com:12051/dgx/bmc/entity-manager.git;protocol=ssh;branch=develop file://blocklist.json"
SRCREV = "1cfa102bdb0435ade263b3805a2cf3457a054f10"

SRC_URI:append = " file://HMC.json \
                   file://blacklist.json \
                   file://hgxb_fpga_chassis.json \
                   file://hgxb_gpu_chassis.json \
                   file://hgxb_pcieretimer_chassis.json \
                   file://hgxb_nvlink_chassis.json \
                   file://hgxb_cx7_chassis.json \
                   file://HGXB_NVLink_Mapping.xlsx \
                   file://nvLink_topology_processor.py \
                   file://hgxb_static_inventory.json \
                   file://hgxb_instance_mapping.json \
                   file://hgxb_gpu_configuration.json \
                   file://hgxb_erot_configuration.json \
                   file://hgxb_erot_bmc0_chassis.json \
                   file://hgxb_erot_fpga0_chassis.json \
                   file://hgxb_erot_cx7_chassis.json \
                   file://hgxb_erot_qm3_chassis.json \
                 "

RDEPENDS:${PN} = " \
        fru-device \
        "

do_configure:append() {
    python3 ${WORKDIR}/nvLink_topology_processor.py ${WORKDIR}/HGXB_NVLink_Mapping.xlsx ${WORKDIR}/hgxb_link_topology.json
}

do_install:append() {
     # Remove unnecessary config files. EntityManager spends significant time parsing these.
     rm -f ${D}/usr/share/entity-manager/configurations/*.json

     install -m 0444 ${WORKDIR}/HMC.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/blacklist.json ${D}/usr/share/entity-manager/
     install -m 0444 ${WORKDIR}/hgxb_fpga_chassis.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_gpu_chassis.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_pcieretimer_chassis.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_nvlink_chassis.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_cx7_chassis.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_link_topology.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_static_inventory.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_instance_mapping.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_gpu_configuration.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_erot_configuration.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_erot_fpga0_chassis.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_erot_bmc0_chassis.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_erot_cx7_chassis.json ${D}/usr/share/entity-manager/configurations
     install -m 0444 ${WORKDIR}/hgxb_erot_qm3_chassis.json ${D}/usr/share/entity-manager/configurations
}

```

#### Step2: Write Configuration Files

##### STATIC INVENTORY AND DYNAMIC INVENTORY
NSM supports static inventory creation. Static inventory means that inventory objects will always be populated to D-Bus no matter if the communication of the NSM Device is ready or not.

The properties of PDIs will be initialized to default value and they should  be updated to correct value once the communication of NSM Device is back(e.g. power on).

What inventory object should be created by nsmd can be configurable by EM json file and whether inventory is static or dynamic is also configured through EM json file or by “probe” property of EM json with more specific.

For static Inventory, the EM json probe rule should not be the condition depending on if EID enumerated or not. And the UUID property of every config PDIs should be in the format, “DEVICE_TYPE=X:INSTANCE_ID=Y” for nsmd to match the config PDI to correct nsmDevice when the device is enumerated.

For dynamic inventory, the EM json probe rule should be "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': X})" for the condition when nsmd created FruDevice PDI for the enumerated EID. And the UUID property of every config PDIs should be “$UUID”.

Static inventory configuration : https://gitlab-master.nvidia.com/dgx/bmc/openbmc/-/blob/develop/meta-nvidia/meta-hgxb/recipes-phosphor/configuration/entity-manager/files/hgxb_static_inventory.json

e.g.

```
{
    "Exposes": [],
    "Name": "NSM_DEV_GPU_0",
    "Probe": "TRUE",
    "Type": "NSM_Configs",
    "xyz.openbmc_project.NsmDevice": {
        "DEVICE_TYPE": 0,
        "INSTANCE_NUMBER": 0,
        "CONNECTED_RETIMER_INSTANCE_NUM": 0,
        "UUID": "STATIC:0:0"
}

```

dynamic inventory : https://gitlab-master.nvidia.com/dgx/bmc/openbmc/-/blob/develop/meta-nvidia/meta-hgxb/recipes-phosphor/configuration/entity-manager/files/hgxb_gpu_chassis.json

```
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
        }
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

```

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
Eid: 34 --> Mocks Switch with instanceId 6 [as per mapping eid 34 should have 0 as instanceID]
Eid: 35 --> Mocks Switch with instanceId 8 [as per mapping eid 35 should have 1 as instanceID]
Eid: 36 --> Mocks PCIeBridge with instanceId 6 [as per mapping uuid "24000000-0000-0000-0000-000000000000" should have 0 as instanceID]
```

```
    Sample EM josn:
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
      {
        "Name": "SwitchMapping",
        "Type": "NSM_GetInstanceIDByDeviceEID",
        "MappingArr": [
          34,
          35
        ]
      },
      {
        "Name": "PCIeBridgeMapping",
        "Type": "NSM_GetInstanceIDByDeviceUUID",
        "MappingArr": [
          "24000000-0000-0000-0000-000000000000"
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
        | |- /xyz/openbmc_project/inventory/system/chassis/HGX_FPGA_0
........................................
        `- /xyz/openbmc_project/inventory/system/nsm_configs
          |- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping
          | |- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/GPUMapping
          | |- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/PCIeBridgeMapping
          | `- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/SwitchMapping
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_CX_0
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_FPGA_0
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_0
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_1
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_2
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_3
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_4
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_5
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_6
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_7
          |- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_QM_0
          `- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_QM_1
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

root@hgxb:~# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/PCIeBridgeMapping
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
.Name                                             property  s         "PCIeBridgeMapping"                      emits-change
.Type                                             property  s         "NSM_UUIDMapping"                        emits-change

root@hgxb:~# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/SwitchMapping
NAME                                             TYPE      SIGNATURE RESULT/VALUE     FLAGS
org.freedesktop.DBus.Introspectable              interface -         -                -
.Introspect                                      method    -         s                -
org.freedesktop.DBus.Peer                        interface -         -                -
.GetMachineId                                    method    -         s                -
.Ping                                            method    -         -                -
org.freedesktop.DBus.Properties                  interface -         -                -
.Get                                             method    ss        v                -
.GetAll                                          method    s         a{sv}            -
.Set                                             method    ssv       -                -
.PropertiesChanged                               signal    sa{sv}as  -                -
xyz.openbmc_project.Configuration.NSM_GetInstan  interface -         -                -
.MappingArr                                      property  at        2 34 35          emits-change
.Name                                            property  s         "SwitchMapping"  emits-change
.Type                                            property  s         "NSM_GetInstance emits-change
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
          | |- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/PCIeBridgeMapping
          | `- /xyz/openbmc_project/inventory/system/nsm_configs/Mapping/SwitchMapping
          `- /xyz/openbmc_project/inventory/system/nsm_configs/NSM_DEV_GPU_0
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
