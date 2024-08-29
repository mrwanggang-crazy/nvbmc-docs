# SEL for BMC WEBUI

Author: Sean Zhang

Created: Augest 27, 2024

## Problem Description
Currently, the web UI just show the event logs but not SEL logs, since SEL logs should contain more information like sensor and asserted/deasserted status, user want to have that page to show all the informations for SEL logs. To support this, more fields need to be added to the Redfish SEL response which are already defined in the schema.

## Background and References

1. https://redfish.dmtf.org/schemas/v1/LogEntry.v1_16_1.json

## Requirements

The customers need to use the web UI of BMC to view the information of SEL logs. They want to have a web page to just list the SEL logs rather than event logs to get more information like components(sensors) and asserted/deasserted status same as ‘ipmitool sel list’ output.

## Proposed Design

```
+-------------------------------------------------------------------------------------------------------------------------------------------------+
|                                                                                                                                                 |
|   +-----------------------+                                                                                                                     |
|   | SEL Generator         |                                                                                                                     |
|   |  phosphor-host-ipmid, |                                                                                                                     |
|   |  dpu-manager, etc     |                                                                                                                     |
|   +-----------+-----------+                                                                                                                     |
|               |                                                                                                                                 |
|   Generate    | xyz.openbmc_project.Logging.IPMI                                                                                                |
|   SEL Event   v         IpmiSelAdd                                                                                                              |
|   +-----------+-----------+                         +---------------------------+                +----------------+                             |
|   |                       |   Create SEL Entry      |   SEL Dbus Object         |                | BMCWeb         |                             |
|   |  phosphor-sel-logger  +----------Dbus---------->+                           |                |                |                             |
|   |                       |  xyz.openbmc_project.   | Properties                |                |  SEL Entries:  |                             |
|   +-----------+-----------+  Logging.Create         |     AdditionalData        |                |   ...          |                             |
|               |                                     |        +SENSOR_TYPE       |                |   SensorType   |                             |
|  Get Sensor   |    getSensorTypeFromPath            |        +SENSOR_NUMBER     |                |   SensorNumber |                             |
|  Information call  getSensorNumberFromPath          |                           |                |   EntryCode    |                             |
|               |                                     |                           |                |                |               +----------+  |
|               v                                     |                           | Get SEL Entries|                |               |          |  |
| +-------------+-------------+                       |                           +------Dbus----->+                +--Redfish API->+  Web UI  |  |
| |                           |                       |                           |                |                |               |          |  |
| |    phosphor-host-ipmid    |                       |                           |                |                |               +----------+  |
| |                           |                       |                           |                |                |                             |
| |  export methods           |                       |                           |                |                |                             |
| |    getSensorTypeFromPath  |                       |                           |                |                |                             |
| |    getSensorNumberFromPath|                       |                           |                |                |                             |
| +---------------------------+                       |                           |                |                |                             |
|                                                     |                           |                |                |                             |
|                                                     +---------------------------+                +----------------+                             |
|                                                                                                                                                 |
+-------------------------------------------------------------------------------------------------------------------------------------------------+
```
1.  Add SENSOR_TYPE and SENSOR_NUMBER in phosphor-dbus-interfaces/yaml/xyz/openbmc_project/Logging/SEL.metadata.yaml

  - name: Created
    level: INFO
    meta:
        ...
        - str: "SENSOR_TYPE=%s"
          type: string
        - str: "SENSOR_NUMBER=%u"
          type: uint8

2.  Export sdrutils.cpp in phosphor-host-ipmid so that phosphor-sel-logger can call getSensorTypeFromPath and getSensorNumberFromPath to get the sensor information.

dbus_sdr_lib = library(
  'dbus_sdr',
  ['sdrutils.cpp', 'sensorutils.cpp'],
  implicit_include_directories: false,
  dependencies: dbus_sdr_pre,
  version: meson.project_version(),
  install: true,
  install_dir: get_option('libdir'),
  override_options: ['b_lundef=false'])

import('pkgconfig').generate(
  dbus_sdr_lib,
  name: 'libdbussdr',
  version: meson.project_version(),
  description: 'libdbussdr')


  Add getSensorTypeStringPath(const std::string& path) function to convert sensorType with integer value to string with below mapping.

  static constexpr const std::array<const char*, 45> sensorTypeString = {
      "Reserved",
      "Temperature", "Voltage", "Current", "Fan",
      "Physical Chassis Security", "Platform Security Violation Attempt", "Processor",
      "Power Supply / Converter", "PowerUnit", "Cooling Device", "Other Units-based Sensor",
      "Memory", "Drive Slot/Bay", "POST Memory Resize",
      "System Firmware Progress", "Event Logging Disabled", "Watchdog",
      "System Event", "Critical Interrupt", "Button/Switch",
      "Module/Board", "Microcontroller/Coprocessor", "Add-in Card",
      "Chassis", "ChipSet", "Other FRU", "Cable/Interconnect",
      "Terminator", "SystemBoot/Restart", "Boot Error",
      "BaseOSBoot/InstallationStatus", "OS Stop/Shutdown", "Slot/Connector",
      "System ACPI PowerState", "Watchdog", "Platform Alert",
      "Entity Presence", "Monitor ASIC/IC", "LAN",
      "Management Subsystem Health", "Battery", "Session Audit",
      "Version Change", "FRU State"};

  std::string getSensorTypeStringPath(const std::string& path)
  {
      uint8_t sensorType = getSensorTypeFromPath(path);
      if (sensorType >= sensorTypeString.size())
      {
          return std::string();
      }
      return sensorTypeString[sensorType];
  }

  For SEL creation function in ipmid `ipmiStorageAddSEL`, also add SENSOR_TYPE and SENSOR_NUMBER to addtional data during creation. 

3.  Add implementation in phosphor-sel-logger for calling the methods in dbus-sdr/sdrutils.hpp to get the sensor information (sensorType/sensorNumber) and then add to the additional data of SEL entry during creating. "SENSOR_TYPE" for sensor type and "SENSOR_NUMBER" for sensor number.

addData["SENSOR_TYPE"] = getSensorTypeStringPath(path);
addData["SENSOR_NUMBER"] = std::to_string(getSensorNumberFromPath(path));

4.  Add implementation in bmcweb when parsing the additional data in dbus for the SEL entries, present the fields "SensorType"/"SensorNumber"/"EntryCode" in response of /redfish/v1/Systems/system/LogServices/SEL/Entries. Here "EntryCode" is for "Assert"/"Deassert" status.

The information getting from the Additional data in SEL entry:
root@dpu-bmc:~# busctl get-property xyz.openbmc_project.Logging /xyz/openbmc_project/logging/entry/12 xyz.openbmc_project.Logging.Entry AdditionalData
as 8 "EVENT_DIR=1" "GENERATOR_ID=32" "RECORD_TYPE=2" "REDFISH_MESSAGE_ARGS=DPU Hard Reset" "REDFISH_MESSAGE_ID=DPU Hard Reset" "SENSOR_DATA=01" "SENSOR_PATH=/xyz/openbmc_project/state/reboot/sensor" "namespace=SEL" "SENSOR_TYPE=Version Change" "SENSOR_NUMBER=2"

curl https://bmc-ip/redfish/v1/Systems/system/LogServices/SEL/Entries
{
  "@odata.id": "/redfish/v1/Systems/system/LogServices/SEL/Entries",
  "@odata.type": "#LogEntryCollection.LogEntryCollection",
  "Description": "Collection of System Event Log Entries",
  "Members": [
    {
      "@odata.id": "/redfish/v1/Systems/system/LogServices/SEL/Entries/16",
      "@odata.type": "#LogEntry.v1_13_0.LogEntry",
      "Created": "2024-08-20T14:41:39+00:00",
      "EntryType": "SEL",
      "Id": "16",
      "Message": "BMC SW update",
      "MessageId": "BMC SW update",
      "Modified": "2024-08-20T14:41:39+00:00",
      "Name": "System Event Log Entry",
      "Resolved": false,
      "Severity": "OK",
      "EntryCode": "Assert",
      "SensorType": "Version Change",
      "SensorNumber": 2
    }
  ],
  "Members@odata.count": 1,
  "Name": "System Event Log Entries"
}

5. Add page for SEL logs in web UI to get SEL entries and populate them in the SEL entries table.

## Testing
1.	Prepare SEL entry file (sel_entry.txt) for SEL entry creation. Format as below:
SEL_REC_ID TIME_STAMP GEN_ID EvM_Rev SENSOR_TYPE SENSOR_NUM EVENT_DIR EVENT_TYPE EVENT_DATA1 EVENT_DATA2 EVENT_DATA3
2.	Use ‘ipmitool sel add sel_entry.txt’ command to create SEL entry.
3.	Get the SEL entries with Redfish interface (https://{bmc_ip}/redfish/v1/Systems/Bluefield/LogServices/SEL/Entries) and check if the SensorType/SensorNumber/EntryCode fields as expected.
4.	Validate the display in SEL page of web UI.
