# nsmd - Nvidia System Management Daemon

Authors:
 Gilbert Chen (gilbertc@nvidia.com)
 Harshit Aghera (haghera@nvidia.com)
 Rajat Jain (rajatj@nvidia.com)
 Aishwary Joshi (aishwaryj@nvidia.com)
 Utkarsh Yadav (uyadav@nvidia.com)

Primary assignee:
 Harshit Aghera (haghera@nvidia.com)

Reviewers:
 Deepak Kodihalli (dkodihalli@nvidia.com)
 Shakeeb Pasha (spasha@nvidia.com)
 Gilbert Chen (gilbertc@nvidia.com)

## Capabilities
The nsmd service can discover NSM endpoint, gather telemetry data from the endpoints, and can publish them to D-Bus or similar IPC services, for consumer services like bmcweb.

## Relevant Standard Specifications
1. [NVIDIA System Management API Specification](https://nvidia.sharepoint.com/sites/MCTPSystemManagementAPI/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2FMCTPSystemManagementAPI%2FShared%20Documents%2FSpecifications%20%28working%20copy%29&viewid=2378e471%2Dba70%2D4475%2Dabde%2Db3ac9365091f&noAuthRedirect=1)
2. [DMTF MCTP Base Specification](https://www.dmtf.org/dsp/DSP0236)

## Architecture

### nsmd Functional Sequence Diagram
nsmd operational logic is summarised Sequence Diagram given below.
```
           ┌─────────┐             ┌────────────────┐        ┌────────────────┐  ┌───────────────────┐ ┌───────────┐ ┌──────────┐ ┌────────────┐
           │  nsmd   │             │ Entity-Manager │        │    Gpu-Mgr     │  │MCTP Control Daemon│ │MCTP Demux │ │NSM Device│ │ObjectMapper│
           └────┬────┘             └───────┬────────┘        └───────┬────────┘  └────────┬──────────┘ └─────┬─────┘ └─────┬────┘ └──────┬─────┘
                │                          │                         │                    │                  │             │             │
                │                          │                         │                    │                  │             │             │
                │ GetSubTree               │                         │                    │                  │             │             │
                │ /xyz/openbmc_project/inventory                     │                    │                  │             │             │
                │ xyz.openbmc_project.inventory.item.Switch          │                    │                  │             │             │
                | xyz.openbmc_project.inventory.item.FabricAdapter   │                    │                  │             │             │
               ┌┴┐xyz.openbmc_project.inventory.item.Accelerator     │                    │                  │             │             │
 Get Inventory │ ├─────────────────────────┬─────────────────────────┼────────────────────┼──────────────────┼─────────────┼────────────►│
               │ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               │ │   response              │                         │                    │                  │             │             │
               │ │ ◄───────────────────────┼─────────────────────────┼────────────────────┼──────────────────┼─────────────┼─────────────┤
               └┬┘                         │                         │                    │                  │             │             │
               ┌┴┐ ◄───────────────────────┼─────────────────────────┤                    │                  │             │             │
               │ │ ◄───────────────────────┤                         │                    │                  │             │             │
Get Config PDI │ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               └┬┘                         │                         │                    │                  │             │             │
                │                          │                         │                    │                  │             │             │
               ┌┴┐                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
  Create sensor│ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               └┬┘                         │                         │                    │                  │             │             │
                │                          │                         │                    │                  │             │             │
               ┌┴┐                         │                         │                    │                  │             │             │
      Associate│ │                         │                         │                    │                  │             │             │
      Sensor   │ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               └┬┘                         │                         │                    │                  │             │             │
                │                          │                         │                    │                  │             │             │
                │ GetSubTree               │                         │                    │                  │             │             │
                │ /xyz/openbmc_project/mctp│                         │                    │                  │             │             │
               ┌┴┐xyz.openbmc_project.MCTP.Endpoint                  │                    │                  │             │             │
  Get EID list │ ├─────────────────────────┬─────────────────────────┼────────────────────┼──────────────────┼─────────────┼────────────►│
               │ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               │ │   response              │                         │                    │                  │             │             │
               │ │ ◄───────────────────────┼─────────────────────────┼────────────────────┼──────────────────┼─────────────┼─────────────┤
               └┬┘                         │                         │                    │                  │             │             │
                │                          │                         │                    │                  │             │             │
               ┌┴┐GetSupportedMessageTypes │                         │                    │                  │             │             │
  Discover NSM │ ├─────────────────────────┼─────────────────────────┼────────────────────┼─────────────────►│             │             │
  Endpoint     │ │                         │                         │                    │                  ├────────────►│             │
               │ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               │ │   response              │                         │                    │                  │◄────────────┤             │
               │ │ ◄───────────────────────┼─────────────────────────┼────────────────────┼──────────────────┤             │             │
               └┬┘                         │                         │                    │                  │             │             │
                │                          │                         │                    │                  │             │             │
               ┌┴┐GetQueryDeviceInformation│                         │                    │                  │             │             │
               │ ├─────────────────────────┼─────────────────────────┼────────────────────┼─────────────────►│             │             │
 GetFRU Data   │ │                         │                         │                    │                  ├────────────►│             │
 And Expose to │ │                         │                         │                    │                  │             │             │
 D-Bus         │ │                         │                         │                    │                  │             │             │
               │ │   response              │                         │                    │                  │◄────────────┤             │
               │ │ ◄───────────────────────┼─────────────────────────┼────────────────────┼──────────────────┤             │             │
               │ │                         │                         │                    │                  │             │             │
               │ │GetInventoryInformation  │                         │                    │                  │             │             │
               │ ├─────────────────────────┼─────────────────────────┼────────────────────┼─────────────────►│             │             │
               │ │                         │                         │                    │                  ├────────────►│             │
               │ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               │ │   response              │                         │                    │                  │◄────────────┤             │
               │ │ ◄───────────────────────┼─────────────────────────┼────────────────────┼──────────────────┤             │             │
               │ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               │ │                         │   ┌─────────────────┐   │                    │                  │             │             │
               │ │ D-Bus signal notify     │   │Config json files│   │                    │                  │             │             │
               │ │ New FruDevice created   │   └────────┬────────┘   │                    │                  │             │             │
               └┬┴────────────────────────┬┴┐           │            │                    │                  │             │             │
                │                         │ │           │            │                    │                  │             │             │
                │          Probe json file│ │           │            │                    │                  │             │             │
                │                         │ │           │            │                    │                  │             │             │
                │                         │ │◄──────────┘            │                    │                  │             │             │
                │                         └┬┘  matched file          │                    │                  │             │             │
                │                          │                         │                    │                  │             │             │
                │                         ┌┴┐                        │                    │                  │             │             │
                │     Create inventory obj│ │                        │                    │                  │             │             │
                │                         │ │                       ┌┴┐                   │                  │             │             │
                │                         │ │                       │ │                   │                  │             │             │
                │ ◄───────────────────────┴┬┘   Create inventory obj│ │                   │                  │             │             │
                │  Send InterfaceAdd signal│                        │ │                   │                  │             │             │
                │                          │                        │ │                   │                  │             │             │
                │ ◄────────────────────────┼────────────────────────┴┬┘                   │                  │             │             │
               ┌┴┐                         │ Send InterfaceAdd signal│                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
  Create sensor│ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               └┬┘                         │                         │                    │                  │             │             │
                │                          │                         │                    │                  │             │             │
               ┌┴┐                         │                         │                    │                  │             │             │
      Associate│ │                         │                         │                    │                  │             │             │
      Sensor   │ │                         │                         │                    │                  │             │             │
               │ │                         │                         │                    │                  │             │             │
               └┬┘                         │                         │                    │                  │             │             │
  ┌────────────►│                          │                         │                    │                  │             │             │
  │            ┌┴┐ GetPortTelemetryCounter │                         │                    │                  │             │             │
  │ Poll sensor│ ├─────────────────────────┼─────────────────────────┼────────────────────┼─────────────────►│             │             │
  │ State      │ │                         │                         │                    │                  ├────────────►│             │
  │            │ │                         │                         │                    │                  │             │             │
  │            │ │                         │                         │                    │                  │             │             │
  │            │ │                         │                         │                    │                  │             │             │
  │            │ │  response               │                         │                    │                  │◄────────────┤             │
  │            │ │◄────────────────────────┼─────────────────────────┼────────────────────┼──────────────────┤             │             │
  │            └┬┘                         │                         │                    │                  │             │             │
  │             │                          │                         │                    │                  │             │             │
  └─────────────┤                          │                         │                    │                  │             │             │
   Timer loop   │                          │                         │                    │                  │             │             │
```

### End to End data path of OpenBMC service block diagram
```
                    ┌──────────────────┐
                    │    Redfish       │
                    └──────┬───────────┘
    Async DBus Calls       │   ▲
                           ▼   │
                    ┌──────────┴───────┐
                    │      D-Bus       │
                    └──────┬───────────┘
       D-Bus req/Res       │  ▲
                           ▼  │
         ┌────────────────────┴───────────────────┐
         │  NSMD                                  │
         │ ┌──────────┐ ┌──────────┐┌──────────┐  │
         │ │coroutine1│ │coroutine1││coroutine1│  │
         │ └──────────┘ └──────────┘└──────────┘  │
         │                                        │
         └──────┬───────────┬────────────┬────────┘
  Unix Socket   │  ▲        │  ▲         │  ▲
                ▼  │        ▼  │         ▼  │
              ┌────┴───────────┴────────────┴──┐
              │ MCTP demux daemon              │
              └─┬───────────┬────────────┬─────┘
MCTP over PCIe  │  ▲        │  ▲         │  ▲
                ▼  │        ▼  │         ▼  │
              ┌────┴───────────┴────────────┴──┐
              │       FPGA                     │
              └─┬────────────┬───────────┬─────┘
                │  ▲         │  ▲        │  ▲
 MCTP over I2C  ▼  │         ▼  │        ▼  │
              ┌────┴──┐    ┌────┴─┐    ┌────┴──┐
              │  CX7  │    │ QM3  │    │ GB100 │
              └───────┘    └──────┘    └───────┘
```

## Design

## Interaction with other services and relevant D-Bus APIs
nsmd interacts with other services listed below, using D-Bus IPC mechanism. In OpenBMC framework D-Bus Interfaces sometimes are referred as Phosphor D-Bus Interfaces (or PDI for short), and hence both the terms are used interchangeably in this document.
1. MCTP demux and control daemons
2. Entity Manager
3. PLDM daemon

## Platform Enablement
Following sections outline steps to be followed to enable telemetry acquisition from an NSM endpoint using nsmd. In addition these sections can also be referred to understand various nsmd capabilities and its dependencies on other services and involved D-Bus Interfaces.

### Discovering NSM endpoint
nsmd detects creation of D-Bus Interface xyz.openbmc_project.MCTP.Endpoint at object path /xyz/openbmc_project/mctp. To identify whether an MCTP endpoint support NSM, nsmd check if 0x7E (VDM-PCI) is present in Property SupportedMessageTypes of this Interface. Similar exercise can also be carried out manually, to check whether an MCTP endpoint supports NSM or not.

nsmd uses D-Bus IPC service, to publish information for each device endpoints that supports NSM. D-Bus Interface - herein referred as FRU PDI - xyz.openbmc_project.FruDevice will be published at object path /xyz/openbmc_project/FruDevice/{DeviceType}_{InstanceNumber} by nsmd.


#### FRU Device PDI Properties

#### FRU Device PDI Properties
List of Properties of FRU Device PDI created by nsmd. The list is not exhaustive.

| Property               	| Type   	| Mandatory/Optional | NSM Command used to get Value            | Use                         |
|--------------------------	|----------	|------------------- |----------------------------------------- | --------------------------- |
| BOARD_PART_NUMBER       	| string 	| Mandatory          | Type 3 Get Inventory Information (0x11)  | For debugability.           |
| DEVICE_TYPE            	| byte  	| Mandatory          | Type 0 Query Device Identification (0x09)| To determine list of inventories to be published. |
| INSTANCE_NUMBER          	| byte   	| Mandatory          | Type 0 Query Device Identification (0x09)| To determine list of inventories to be published. |
| SERIAL_NUMBER            	| string   	| Mandatory          | Type 3 Get Inventory Information (0x11)  | For debugability.           |
| UUID                  	| string   	| Mandatory          | NA (Populated by MCTP Control Daemon)    | To uniquely identify a device and EID lookup.     |


#### Example 1
FruDevice D-Bus object (for exposition purpose only)
```
root@e4869:~# busctl introspect xyz.openbmc_project.NSM /xyz/openbmc_project/FruDevice/30
NAME                                TYPE      SIGNATURE RESULT/VALUE         FLAGS
xyz.openbmc_project.FruDevice       interface -         -                    -
.BOARD_PART_NUMBER                  property  s         "MCX750500B-0D00_DK" emits-change
.DEVICE_TYPE                        property  y         2                    emits-change
.EID                                property  y         30                   emits-change
.INSTANCE_NUMBER                    property  y         0                    emits-change
.SERIAL_NUMBER                      property  s         "SN123456789"        emits-change
.UUID                               property  s         "550e8400-e29b-41d4- emits-change
```

### Enabling an NSM endpoint and gathering of telemetry values from it
nsmd looks out for creation of certain D-Bus Interfaces at /xyz/openbmc_project/inventory Object path, to start requesting telemetry value from an NSM endpoint.

These interfaces should also contains individual telemetry specific details like which NSM command to use and its content. These interfaces are herein referred to as Configuration PDIs (Phosphor D-Bus Interfaces).

However, nsmd doesn't create Configuration PDIs itself and rely on Entity Manager for their creation. nsmd has configuration driven design and since Configuration PDIs are static, Entity Manager is available as part of OpenBMC infrastructure to host Configuration PDIs on D-Bus. Entity Manager detects the existence of D-Bus Interface xyz.openbmc_project.FruDevice (could have been published by nsmd as mentioned in previous section), on any of the D-Bus services and publishes Configuration PDI based on its content. Entity Manager uses JSON configuration file to determine list of Configuration PDI and their content to be published, upon detection of certain type of FRU Device. Please refer Entity Manager design documents for in depth understanding of the process of publishing Configuration PDI from FRU device D-Bus Interface.

### Structure for EM configs for NSM with examples to follow
As of now NSM support following devices:
```
typedef enum {
	NSM_DEV_ID_GPU = 0,
	NSM_DEV_ID_SWITCH = 1,
	NSM_DEV_ID_PCIE_BRIDGE = 2,
	NSM_DEV_ID_BASEBOARD = 3,
	NSM_DEV_ID_UNKNOWN = 0xff,
} NsmDeviceIdentification;
```
- Now based on MCTP discovery, for MCTP endpoints which are NSM endpoints, NSM service will create a fruDevice object for each instance of device found.
- We fire a few nsmd commands for FRU and inventory details for the device. We then expose properties required for EM configuration on the PDI.

e.g.
```
:# busctl tree xyz.openbmc_project.NSM
`-/xyz
  `-/xyz/openbmc_project
    `-/xyz/openbmc_project/FruDevice
      `-/xyz/openbmc_project/FruDevice/31
# busctl introspect xyz.openbmc_project.NSM /xyz/openbmc_project/FruDevice/31
NAME                                TYPE      SIGNATURE RESULT/VALUE                           FLAGS
org.freedesktop.DBus.Introspectable interface -         -                                      -
.Introspect                         method    -         s                                      -
org.freedesktop.DBus.Peer           interface -         -                                      -
.GetMachineId                       method    -         s                                      -
.Ping                               method    -         -                                      -
org.freedesktop.DBus.Properties     interface -         -                                      -
.Get                                method    ss        v                                      -
.GetAll                             method    s         a{sv}                                  -
.Set                                method    ssv       -                                      -
.PropertiesChanged                  signal    sa{sv}as  -                                      -
xyz.openbmc_project.FruDevice       interface -         -                                      -
.BOARD_PART_NUMBER                  property  s         "MCX750500B-0D00_DK"                   emits-change
.DEVICE_TYPE                        property  y         2                                      emits-change
.INSTANCE_NUMBER                    property  y         0                                      emits-change
.SERIAL_NUMBER                      property  s         "SN123456789"                          emits-change
.UUID                               property  s         "c13e2b99-68e4-45f1-8686-409009062aa8" emits-change
```

- As we can see CX7 was identified, we created object /xyz/openbmc_project/FruDevice/31 and interface “xyz.openbmc_project.FruDevice” , exposing UUID, DEVICE_TYPE, INTANCE_NUMBER etc on fru device interface.

#### PCIE BRIDGE DEVICE EM config
Here is the basic example for the device pcie bridge EM json. It also contains sensors to be assumed by pldm type 2.
For HGXB - https://gitlab-master.nvidia.com/dgx/bmc/openbmc/-/blob/develop/meta-nvidia/meta-hgxb/recipes-phosphor/configuration/entity-manager/files/hgxb_cx7_chassis.json

```
{
        "Exposes": [
            {
                "Name": "HGX_Driver_NVLinkManagementNIC_$INSTANCE_NUMBER",
                "Type": "NSM_NVLinkManagementSWInventory",
                "UUID": "$UUID",
                "Manufacturer": "Nvidia",
                "Priority": false
            },
            {
                "Name": "HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Temp_0",
                "Type": "SensorAuxName",
                "SensorId": 1,
                "AuxNames": ["HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Temp_0"]
            },
            {
                "Name": "HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Port_0_Temp_0",
                "Type": "SensorAuxName",
                "SensorId": 8,
                "AuxNames": ["HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Port_0_Temp_0"]
            },
            {
                "Name": "HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Port_1_Temp_0",
                "Type": "SensorAuxName",
                "SensorId": 9,
                "AuxNames": ["HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Port_1_Temp_0"]
            }
        ],
        "Probe": "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': 2})",
        "Name": "NVLinkManagementNIC_$INSTANCE_NUMBER",
        "Type": "NetworkAdapters",
        "Parent_Chassis": "/xyz/openbmc_project/inventory/chassis/Baseboard_0",
        "xyz.openbmc_project.Inventory.Decorator.Asset": {
            "Manufacturer": "Nvidia",
            "Model": "$BOARD_PRODUCT_NAME",
            "PartNumber": "$BOARD_PART_NUMBER",
            "SerialNumber": "$BOARD_SERIAL_NUMBER"
        },
        "xyz.openbmc_project.Inventory.Item.Chassis": {
            "Type": "xyz.openbmc_project.Inventory.Item.Chassis.ChassisType.Component"
        },
        "xyz.openbmc_project.Common.UUID": {
            "UUID": "$UUID"
        },
        "xyz.openbmc_project.Inventory.Item.NetworkInterface": {},
        "xyz.openbmc_project.Inventory.Decorator.Instance": {
            "InstanceNumber": "$INSTANCE_NUMBER"
        }
     },
```
- As soon as the probe gets true , we expose 1 sensor here, for cx7 software inventory related to the driver version.
- “Why UUID”: This uuid will be passed on from fru interface on device objects in NSM. It is required because UUID will be used to uniquely identify the EID/device we are running the nsmd command for. EID is not unique , may change across restarts, after dropping from mctp network and rediscover  etc.
- “Significance of Priority” -  It reflects that the sensor is dynamic. Needto be updated in polling coroutine. Now it has value true, its put in priority sensor list, if its false it is put in round robin list.

On Entity Manager we have:

```
`-/xyz/openbmc_project/inventory/system/networkadapters
          `-/xyz/openbmc_project/inventory/system/networkadapters/NVLinkManagementNIC_0
|-/xyz/openbmc_project/inventory/system/networkadapters/NVLinkManagementNIC_0/HGX_Driver_NVLinkManagementNIC_0
```

```
root@umbriel:~# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/networkadapters/NVLinkManagementNIC_0/HGX_Driver_NVLinkManagementNIC_0NAME                                                              TYPE      SIGNATURE RESULT/VALUE                           FLAGS
org.freedesktop.DBus.Introspectable                               interface -         -                                      -
.Introspect                                                       method    -         s                                      -
org.freedesktop.DBus.Peer                                         interface -         -                                      -
.GetMachineId                                                     method    -         s                                      -
.Ping                                                             method    -         -                                      -
org.freedesktop.DBus.Properties                                   interface -         -                                      -
.Get                                                              method    ss        v                                      -
.GetAll                                                           method    s         a{sv}                                  -
.Set                                                              method    ssv       -                                      -
.PropertiesChanged                                                signal    sa{sv}as  -                                      -
xyz.openbmc_project.Configuration.NSM_NVLinkManagementSWInventory interface -         -                                      -
.Manufacturer                                                     property  s         "Nvidia"                               emits-change
.Name                                                             property  s         "HGX_Driver_NVLinkManagementNIC_0"     emits-change
.Priority                                                         property  b         false                                  emits-change
.Type                                                             property  s         "NSM_NVLinkManagementSWInventory"      emits-change
.UUID                                                             property  s         "a007f776-7805-4e00-0000-0000482e4f00" emits-change
```

After NSMD consumes it:
- It creates /xyz/openbmc_project/inventory_software/HGX_Driver_NVLinkManagementNIC_0 object path which contains sensor information for driver version.
- It is kept in priority round robin polling loop because priority property was false in EM config.

```
`-/xyz
  `-/xyz/openbmc_project
    |-/xyz/openbmc_project/FruDevice
    | `-/xyz/openbmc_project/FruDevice/31
    `-/xyz/openbmc_project/inventory_software
      `-/xyz/openbmc_project/inventory_software/HGX_Driver_NVLinkManagementNIC_0
```

```
root@umbriel:~# busctl introspect xyz.openbmc_project.NSM /xyz/openbmc_project/inventory_software/HGX_Driver_NVLinkManagementNIC_0
NAME                                                  TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.freedesktop.DBus.Introspectable                   interface -         -                                        -
.Introspect                                           method    -         s                                        -
org.freedesktop.DBus.Peer                             interface -         -                                        -
.GetMachineId                                         method    -         s                                        -
.Ping                                                 method    -         -                                        -
org.freedesktop.DBus.Properties                       interface -         -                                        -
.Get                                                  method    ss        v                                        -
.GetAll                                               method    s         a{sv}                                    -
.Set                                                  method    ssv       -                                        -
.PropertiesChanged                                    signal    sa{sv}as  -                                        -
xyz.openbmc_project.Association.Definitions           interface -         -                                        -
.Associations                                         property  a(sss)    0                                        emits-change writable
xyz.openbmc_project.Inventory.Decorator.Asset         interface -         -                                        -
.BuildDate                                            property  s         ""                                       emits-change writable
.Manufacturer                                         property  s         "a0f7f076-7805-4200-0000-0000482e4300"   emits-change writable
.Model                                                property  s         ""                                       emits-change writable
.Name                                                 property  s         ""                                       emits-change writable
.PartNumber                                           property  s         ""                                       emits-change writable
.SKU                                                  property  s         ""                                       emits-change writable
.SerialNumber                                         property  s         ""                                       emits-change writable
.SparePartNumber                                      property  s         ""                                       emits-change writable
.SubModel                                             property  s         ""                                       emits-change writable
xyz.openbmc_project.Software.Version                  interface -         -                                        -
.Purpose                                              property  s         "xyz.openbmc_project.Software.Version... emits-change writable
.SoftwareId                                           property  s         ""                                       emits-change writable
.Version                                              property  s         "MockDriverVersion 1.0.0"                emits-change writable
xyz.openbmc_project.State.Decorator.OperationalStatus interface -         -                                        -
.Functional                                           property  b         true                                     emits-change writable
.State                                                property  s         "xyz.openbmc_project.State.Decorator.... emits-change writable
```

This is the general pattern we follow for sensor creation. For nsmd we are assuming everything as a sensor.

```
|                              STATIC SENSORS                                    |               DYNAMIC SENSORS             |
|--------------------------------------------------------------------------------|-------------------------------------------|
| No nsm command trigger required.   |   NSM command need to be triggered once.  |    Priority           |     Round Robin   |
| E.g. Manufacturer NVIDIA           |                                           |                       |                   |
|                                    |                                           |                       |                   |
|                                    |                                           |                       |                   |
|-----------------------------------------------------------------------------------------------------------------------------
|                      Separate update function()                                |      Separate polling coroutine           |
```

#### GB100 DEVICE EM config

For hgxb:  https://gitlab-master.nvidia.com/dgx/bmc/openbmc/-/blob/develop/meta-nvidia/meta-hgxb/recipes-phosphor/configuration/entity-manager/files/hgxb_gpu_chassis.json

```
{
        "Exposes": [
            {
                "Name": "GPU_$INSTANCE_NUMBER + 1 Processor",
                "Type": "NSM_Processor",
                "UUID": "$UUID",
                "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
                "MIGMode": {
                    "Type": "NSM_MIG",
                    "UUID": "$UUID",
                    "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
                    "Priority": false
                },
                "ECCMode": {
                    "Type": "NSM_ECC",
                    "UUID": "$UUID",
                    "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
                    "Priority": false
                }
            }
        ],
        "Probe": "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': 0})",
        "Name": "GPU_$INSTANCE_NUMBER + 1",
        "Type": "Processor",
        "Parent_Chassis": "/xyz/openbmc_project/inventory/chassis/Baseboard_0",
        "xyz.openbmc_project.Inventory.Decorator.Asset": {
            "Manufacturer": "Nvidia",
            "Model": "$BOARD_PRODUCT_NAME",
            "PartNumber": "$BOARD_PART_NUMBER",
            "SerialNumber": "$BOARD_SERIAL_NUMBER"
        },
        "xyz.openbmc_project.Inventory.Decorator.Instance": {
            "InstanceNumber": "$INSTANCE_NUMBER"
        }
    }
```

- Here we handle both scenario whether we want to index gpu from 0 or 1 .
- For hgxb it is 1 based.

```
root@umbriel:/usr/share/entity-manager/configurations# busctl tree xyz.openbmc_project.EntityManager
`- /xyz
`- /xyz/openbmc_project
|- /xyz/openbmc_project/EntityManager
`- /xyz/openbmc_project/inventory
`- /xyz/openbmc_project/inventory/system
|- /xyz/openbmc_project/inventory/system/fabric
| `- /xyz/openbmc_project/inventory/system/fabric/Fabric_0
| `- /xyz/openbmc_project/inventory/system/fabric/Fabric_0/QM3_0
`- /xyz/openbmc_project/inventory/system/processor
`- /xyz/openbmc_project/inventory/system/processor/GPU_1
`- /xyz/openbmc_project/inventory/system/processor/GPU_1/GPU_1_Processor
```

```
root@umbriel:/usr/share/entity-manager/configurations# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/processor/GPU_1/GPU_1_Processor
NAME                                                              TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.freedesktop.DBus.Introspectable                               interface -         -                                        -
.Introspect                                                       method    -         s                                        -
org.freedesktop.DBus.Peer                                         interface -         -                                        -
.GetMachineId                                                     method    -         s                                        -
.Ping                                                             method    -         -                                        -
org.freedesktop.DBus.Properties                                   interface -         -                                        -
.Get                                                              method    ss        v                                        -
.GetAll                                                           method    s         a{sv}                                    -
.Set
                    method    ssv       -                                        -
.PropertiesChanged                                                signal    sa{sv}as  -                                        -
xyz.openbmc_project.Configuration.NSM_Processor                   interface -         -                                        -
.InventoryObjPath                                                 property  s         "/xyz/openbmc_project/inventory/syste... emits-change
.Name                                                             property  s         "GPU_1 Processor"                        emits-change
.Type                                                             property  s         "NSM_Processor"                          emits-change
.UUID                                                             property  s         "c13e2b99-68e4-45f1-8686-409009062aa8"   emits-change
xyz.openbmc_project.Configuration.NSM_Processor.ECCMode           interface -         -                                        -
.InventoryObjPath                                                 property  s         "/xyz/openbmc_project/inventory/syste... emits-change
.Priority                                                         property  b         false                                    emits-change
.Type                                                             property  s         "NSM_ECC"                                emits-change
.UUID                                                             property  s         "c13e2b99-68e4-45f1-8686-409009062aa8"   emits-change
xyz.openbmc_project.Configuration.NSM_Processor.EDPpScalingFactor interface -         -                                        -
.InventoryObjPath                                                 property  s         "/xyz/openbmc_project/inventory/syste... emits-change
.Priority                                                         property  b         false                                    emits-change
.Type                                                             property  s         "NSM_EDPp"                               emits-change
.UUID                                                             property  s         "c13e2b99-68e4-45f1-8686-409009062aa8"   emits-change
xyz.openbmc_project.Configuration.NSM_Processor.MIGMode           interface -         -                                        -
.InventoryObjPath                                                 property  s         "/xyz/openbmc_project/inventory/syste... emits-change
.Priority                                                         property  b         false                                    emits-change
.Type                                                             property  s         "NSM_MIG"                                emits-change
.UUID                                                             property  s         "c13e2b99-68e4-45f1-8686-409009062aa8"   emits-change
```

- Now if we compare EM config and the results we can see that main configuration PDI is "xyz.openbmc_project.Configuration.NSM_Processor".
- ECCMode, MIGMode which are created in sub blocks of EM json. here are created as kind of secondary PDI's, e.g. "xyz.openbmc_project.Configuration.NSM_Processor.ECCMode" etc.

#### PCIeRetimer DEVICE EM config

For hgxb: https://gitlab-master.nvidia.com/dgx/bmc/openbmc/-/blob/develop/meta-nvidia/meta-hgxb/recipes-phosphor/configuration/entity-manager/files/hgxb_pcieretimer_chassis.json

- This is a special scenario. Retimer is not a device which is directly supported by nsmd. We get all its info from FPGA.
- So here we tightly couple retimer with fpga EM json.
- As soon as fpga is up we create all retimers supported.

```
{
        "Exposes": [
            {
                "Name": "HGX_PCIeRetimer_0",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 0,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_0_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_0"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_1",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 1,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_1_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_1"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_2",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 2,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_2_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_2"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_3",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 3,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_3_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_3"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_4",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 4,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_4_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_4"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_5",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 5,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_5_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_5"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_6",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 6,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_6_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_6"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_7",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 7,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_7_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_7"
                ]
            }
        ],
        "Probe": "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': 3})",
        "Name": "HGX_FPGA_0",
        "Type": "fpga",
        "Parent_Chassis": "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
    }
```

- Each retimer has type "NSM_PCIeRetimer".
- the advantage of this is future exposes on all pcieretimer now can we done wirth single json block.

e.g. Look at the probe, it gets true for all retimer devices.
```
{
        "Exposes": [
            {
                "Name": "HGX_FW_PCIeRetimer_$INSTANCE_NUMBER",
                "Type": "NSM_PCIeRetimer_FWInventory",
                "Association": [
                    "inventory",
                    "activation",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_PCIeRetimer_$INSTANCE_NUMBER",
                    "software_version",
                    "updateable",
                    "/xyz/openbmc_project/software"
                ],
                "UUID": "$UUID",
                "Manufacturer": "Nvidia",
                "INSTANCE_NUMBER": "$INSTANCE_NUMBER"
            }
        ],
        "Probe": "xyz.openbmc_project.Configuration.NSM_PCIeRetimer({})",
        "Name": "PCIeRetimer_$INSTANCE_NUMBER",
        "Type": "chassis",
        "Parent_Chassis": "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
    }
```

#### HSC Device
- Device for which no chassis schema is applicable.

```
{
           "Name": "HGX_Chassis_0_HSC_0_Temp_0",
           "Type" : "NSM_Temp",
           "Associations": [
               {
                   "Forward" : "chassis",
                   "Backward" : "all_sensors",
                   "AbsolutePath" : "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
               }
           ],
           "UUID": "$UUID",
           "Aggregated": true,
           "SensorId": 192,
           "Priority": true
        },
{
           "Name": "HGX_Chassis_0_HSC_0_Power_0",
           "Type" : "NSM_Power",
           "Associations": [
               {
                   "Forward" : "chassis",
                   "Backward" : "all_sensors",
                   "AbsolutePath" : "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
               }
           ],
           "UUID": "$UUID",
           "Aggregated": true,
           "SensorId": 128,
           "AveragingInterval": 0,
           "Priority": true
        },
 {
           "Name": "HGX_Chassis_0_StandbyHSC_0_Power_0",
           "Type" : "NSM_Power",
           "Associations": [
               {
                   "Forward" : "chassis",
                   "Backward" : "all_sensors",
                   "AbsolutePath" : "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
               }
           ],
           "UUID": "$UUID",
           "Aggregated": true,
           "SensorId": 138,
           "AveragingInterval": 0,
           "Priority": true
        },
],
      "Probe": "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': 3})",
      "Name": "HGX_FPGA_0",
      "Type": "chassis",
      "Parent_Chassis": "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
      "xyz.openbmc_project.Inventory.Decorator.Asset":
      {
        "Manufacturer": "Nvidia",
        "Model": "$BOARD_PRODUCT_NAME",
        "PartNumber": "$BOARD_PART_NUMBER",
        "SerialNumber": "$BOARD_SERIAL_NUMBER"
      },
      "xyz.openbmc_project.Inventory.Item.Chassis": {
         "Type": "xyz.openbmc_project.Inventory.Item.Chassis.ChassisType.Module"
      },
      "xyz.openbmc_project.Inventory.Decorator.Instance": {
            "InstanceNumber": "$INSTANCE_NUMBER"
      }
   }
```

#### NSM Event Configs

- json blocks are of 2 types
- applicable for all message types for each device.

```
 {
            "Name": "GlobalEventSetting",
            "Type": "NSM_EventSetting",
            "UUID": "$UUID",
            "EventGenerationSetting": 2     #push mode selected
         },
```

- applicable for each messgae type for each device.

```
{
            "Name": "PlatformEnvironmentalEventSetting",
            "Type": "NSM_EventConfig",
            "MessageType": 3,     #messagetype applicable
            "UUID": "$UUID",
            "SubscribedEventIDs": [ #events to subscribe for
               0,
               1
            ],
            "AcknowledgementEventIds": [   #events for ack we want
               0,
               1
            ]
         },
```

For detaisl can be found in events block of this document.

### List of Configuration PDIs of nsmd
TODO - All Configuration PDIs with their application and type and description of each of its property are to be added in this section.


### Steps to enable nsmd for a specific platform
To enable nsmd for a Platform, please follow steps given below.

1. Include nsmd as distro dependency for the Platform.
2. Create Entity Manager configuration file for the device that supports NSM, and configure Entity Manager to use this file for the Platform. Refer Entity Manager documentation for information on involved steps. An example file in given in earlier sections.
3. Provide Platform specific settings like request timeouts, number of retries etc, by using Meson Options (nsmd uses Meson build system). Use EXTRA_OEMESON variable from meson bbclass to provide non-default values for these settings. Refer meson_options.txt file at nsmd repo for list of all available configuration options.

Example Entity Manager Platform Configuration [file](https://gitlab-master.nvidia.com/dgx/bmc/openbmc/-/tree/develop/meta-nvidia/meta-umbriel/recipes-phosphor/configuration/entity-manager/files?ref_type=heads).

## Reference
1. https://www.dmtf.org/sites/default/files/standards/documents/DSP0257_1.0.1_0.pdf

2. https://www.dmtf.org/sites/default/files/standards/documents/DSP0236_1.3.0.pdf

3. https://www.dmtf.org/sites/default/files/standards/documents/DSP0249_1.1.0.pdf