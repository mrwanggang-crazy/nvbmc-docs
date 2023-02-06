### Set Glacier time from BMC/HMC

Author: Tom Joseph

Created: Feb 3, 2023

#### Problem Description

BMC/HMC has downstream devices like Glacier, Glacier's internal logs are
collected for debugging. If the time of BMC/HMC is not included in Glacier logs
it is not possible to correlate the events in the system with the events in 
Glacier. The expectation is that BMC/HMC provides its timestamp to
Glacier and Glacier internal logs include the external timestamp. This feature
applies for both BMC and HMC, so BMC and HMC can be used interchangeably in this
document.

#### Background and References

Glacier supports MCTP VDM command Add_EXT_Timestamp(0x13) to add external
timestamp. The request data contains the number of microseconds that have 
elapsed from the epoch.

|              | Byte        | Description                                     |
| -------------| ----------- | ------------------------------------------------|
|Request data  | 1-8	        | number of microseconds that have elapsed from   |
|              |             | the epoch (uint64, big endian)                  |
|              |             |                                                 |
|Response Data | 1	        | Completion Code                                 |
|              |             | 0x00 - Success                                  |
|              |             | 0x04 - ERROR_NOT_READY(when rate limit is       |
|              |             |        reached for the MCTP VDM command)        |

#### Requirements

1. Set external timestamp for downstream Glaciers on BMC boot.
2. Set external timestamp for downstream Glaciers when user sets the BMC time.
3. Periodic time sync between BMC and Glacier is not a requirement.

#### Proposed Design

BMC to run an oem-nvidia daemon to set the external timestamp for
downstream Glaciers. It has dependency on the
xyz.openbmc_project.Time.Manager.service. The daemon listens for the following 
D-Bus signals and sets the external timestamp by sending MCTP VDM command to
all the Glaciers.

1. PropertiesChanged - When user sets the BMC time 
   xyz.openbmc_project.Time.Manager service emits the PropertiesChanged signal
   on Elapsed property implemented by xyz.openbmc_project.Time.EpochTime
   interface.

2. InterfaceAdded - When xyz.openbmc_project.Time.Manager service
   starts it reads the time from RTC and initializes the Elapsed property.

The daemon also listens for D-Bus signal to watch for discovery of MCTP
endpoints and sets the external timestamp, if not already set by BMC.

#### Alternatives Considered

None

#### Impacts

No performance impact expected as setting the time on BMC will be a one time
operation and the command is not a long running operation on the Glacier.

#### Testing

1. Set BMC time and ensure Glacier log show the external timestamp with the
   time that is set by BMC.
2. Reboot BMC and ensure that BMC sets the external timestamp on Glacier.
3. Validate setting external timestamp will be restricted if inband update is
   disabled.
4. Verify ratelimiting is supported for set glacier time operation.
5. Validate setting Glacier time on BMC boot has no impact on the PLDM firmware
   inventory discovery by BMC.




