# nsmd - Nvidia System Management Daemon

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