# MCTP Tunneling Daemon

Author: Santosh Puranik <spuranik@nvidia.com>

Created: Aug 1, 2024

## Contents

- [MCTP Tunneling Daemon](#mctp-tunneling-daemon)
  - [Contents](#contents)
  - [Overview](#overview)
  - [Limitations](#limitations)
  - [Pre-requisites](#pre-requisites)
  - [Building and Running](#building-and-running)
  - [Setup Routes (with examples)](#setup-routes-with-examples)
  - [Usage Guide](#usage-guide)

## Overview

The MCTP tunneling daemon is a userspace application that can be used to bridge
the in-kernel MCTP stack to a userspace binding. The solution provided by [NVBMC
libmctp][1] can be used to bridge the in-kernel MCTP stack to a userspace USB
binding via a tunneling device that acts as a raw libmctp binding.

The idea is based on [this][2] LF OpenBMC work-in-progress changeset. While the
LF OpenBMC commit demonstrates how to use the tunneling daemon to bind to a
userspace LPC binding, the NVBMC implementation uses a USB binding based on
[libusb][3].

This document is a guide on how to use the tunneling daemon to route packets via
the MCTP USB userspace binding to a supported MCTP USB endpoint such as an MCU.

## Limitations

Since this tunneling daemon is a work-in-prorgess, there are certain limitations
to be aware of:

- The tunneling service needs to be run on the MCTP requester device.
- Only USB userspace bindings are supported as of now.
- The user of the tunneling daemon needs to be aware of the vendor ID and
  product ID of the MCTP USB endpoint connected on the USB bus. The same need to
  be supplied to the tunneling daemon as command line arguments.
- Only one instance of the daemon can be run at this point. This can be
  extended in the future to multiple instances based on the USB bus and port
  numbers.

## Pre-requisites

The following pre-requisites need to be met in order to use the tunneling daemon
with an in-kernel MCTP stack:

- Enable `COFIG_TUN` and `CONFIG_MCTP` in the kernel `KConfig`.
- Enable the in-kernel MCTP userspace tools from this [distro feature][4].

## Building and Running

In order to build the tunneling daemon, the following meson option needs to be
enabled: `enable-mctp-tun`. libusb (at least v 1.0.4) is a required dependency.
Upon successful build, `mctp-tun` executable would be generated.

The binary can be executed as follows:

`mctp-tun vendor_id=0x0955 product_id=0xFFFF class_id=0x0 -v`

Where:
`vendor_id` is the Vendor ID in the remote endpoint's USB device descriptor.
`product_id` is the Product ID in the remote endpoint's USB device descriptor.
`class_id` is currently unused.
`-v` is an optional argument that can be used to generate verbose Tx/Rx logs
from the tunneling daemon. This is useful for debugging.

## Setup Routes (with examples)

The in-kernel MCTP stack needs to be setup such that it can route packets to the
above tunneling device that is created by the tunneling daemon. One can use the
`mctp` userspace utility (which is installed when enabling the MCTP userspace
tools) to setup routing withing the in-kernel MCTP stack. Here are some
reference commands that should be executed after the tunnel daemon is launched.

```sh
#update local endpoint, ex EID 8
mctp addr add 8 dev tun0
#update remote endpoint id ex EID 12
mctp route add 12 via tun0
#bring up the link for tun0
mctp link set tun0 up
```

Now that routes are setup, applications can communicate as usual to the AF_MCTP
socket and send message to EID 12.

## Usage Guide

[1]: https://github.com/NVIDIA/libmctp
[2]: https://gerrit.openbmc.org/c/openbmc/libmctp/+/59581
[3]: https://libusb.info/
[4]:
    https://github.com/openbmc/openbmc/blob/master/meta-phosphor/conf/distro/include/mctp.inc
