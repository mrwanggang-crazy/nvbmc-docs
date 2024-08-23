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

#### Step2: Write Configuration Files

#### Step3: Check expected D-Bus tree (with all config written)

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
