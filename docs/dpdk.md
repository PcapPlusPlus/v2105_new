---
layout: page
title: DPDK Support
permalink: /docs/dpdk
nav_order: 4
---

# DPDK Support
{: .no_toc }

[<img src="{{ site.baseurl }}/resources/logo-dpdk.png" alt="drawing" width="400"/>](https://www.dpdk.org/)

The Data Plane Development Kit (DPDK) is a set of data plane libraries and network interface controller drivers for fast packet processing, currently managed as an open-source project under the Linux Foundation. The DPDK provides a programming framework for x86, ARM, and PowerPC processors and enables faster development of high speed data packet networking applications (taken from [Wikipedia](https://en.wikipedia.org/wiki/Data_Plane_Development_Kit)).

DPDK provides packet processing in line rate using kernel bypass for a large range of network interface cards. Notice that not every NIC supports DPDK as the NIC needs to support the kernel bypass feature. You can read more about DPDK in [DPDK's web-site](https://www.dpdk.org/) and get a list of supported NICs [here](http://core.dpdk.org/supported/).

As DPDK API is written in C, PcapPlusPlus wraps its main functionality in easy-to-use C++ wrappers which have minimum impact on performance and packet processing rate. In addition it brings DPDK to the PcapPlusPlus framework and APIs so you can use DPDK together with other PcapPlusPlus features such as packet analysis, etc.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What does PcapPlusPlus offer for DPDK?

PcapPlusPlus tries to cover the main functionality of DPDK and its most important and popular features. Here is what PcapPlusPlus offers for DPDK:

- Encapsulation of DPDK's initialization process - both outside and inside of the application - using simple scripts and methods
- A C++ class wrapper for DPDK's packet structure (mbuf) which offers the most commonly used functionality
- A C++ class wrapper that encapsulates a DPDK-controlled NIC (A.K.A DPDK port) for receiving, processing and sending packets using DPDK as well as an interface to retrieve NIC info, status, etc.
- Offload packet capture to multiple worker threads running on different cores + control the load balancing configuration between these workers
- Multi RX/TX queue support
- Detailed statistics about packets being captured and processed
- Seamless integration to other PcapPlusPlus capabilities including: parsing of packets received with DPDK using the various protocol parsers offered in PcapPlusPlus, saving packets to a pcap file, sending crafted/edited packets through a DPDK-controlled interface, packet reassembly, and more.
- An easy-to-use C++ wrapper for [DPDK KNI (Kernel NIC Interface)](https://doc.dpdk.org/guides/prog_guide/kernel_nic_interface.html)

## Set up PcapPlusPlus with DPDK

### Supported DPDK versions

The following DPDK versions are currently supported:

- DPDK 19.11 (LTS)
- DPDK 19.08
- DPDK 19.02
- DPDK 18.11 (LTS)
- DPDK 18.08
- DPDK 18.05
- DPDK 18.02
- DPDK 17.11 (LTS)
- DPDK 17.08
- DPDK 17.05
- DPDK 17.02
- DPDK 16.11 (LTS)

Older and newer versions may not work.

In addition, not all poll-mode drivers (PMDs) were tested, but the majority of them should work. Please report an issue if the PMD you're using isn't working.

The following operating systems and configurations were tested:

- Ubuntu 20.04, 18.04, 16.04 LTS 64bit with kernel version > 3 and gcc version >= 4.8
- CentOS 7.1 64bit with kernel 3.x and gcc 4.8.x

### Download and install DPDK

You can find DPDK installation instructions in [DPDK web-site](http://dpdk.org/download).

Building and installing DPDK is pretty straight-forward and in a nutshell goes like this:

```shell
$ cd /dpdk/source/location
$ make config T=[platform_type_string] && make
```

### PcapPlusPlus configuration for DPDK

toolOnce the DPDK build is completed successfully, run PcapPlusPlus configuration script (`configure-linux.sh`) to build PcapPlusPlus with DPDK. Please refer to the [configuration instructions]({{ site.baseurl }}/docs/install/build-source/linux#configuration) to get more details.

### DPDK initialization with PcapPlusPlus

DPDK has two steps of initialization: one that sets up Linux to run DPDK applications and the other at the application level which initializes DPDK data and memory structures. PcapPlusPlus wraps both of them with easy-to-use interfaces. Please see the following two sections to get more information.

### Initialization before application is run

Several Linux configuration steps are needed to run DPDK applications:

- DPDK uses Linux huge-pages for faster virtual to physical page conversion resulting in better performance. Huge-pages must be set before a DPDK application is run
- DPDK uses a designated kernel module for kernel bypass (`igb_uio.ko`). This module needs to be loaded into the kernel
- One or more NICs should move from Linux control to DPDK control
- For DPDK KNI there is one more kernel to be loaded into the kernel (`rte_kni.ko`)

PcapPlusPlus offers a python script that automatically configures all of the above. The script is located in PcapPlusPlus root directory and is named `setup_dpdk.py`. It is based on the [`dpdk-devbind` tool that ships with DPDK](https://doc.dpdk.org/guides/tools/devbind.html) and extends it with more functionality. The script supports both Python 2.7 and 3+.

This script has 3 modes of operation: `setup` to configure the steps mentioned above, `status` to view the status of all known network interfaces, and `restore` to go back to the original Linux configuration.

{% include alert.html alert-type="tip" content="Note: this script uses another file named `setup_dpdk_settings.dat` to keep information needed for restoring the Linux configuration. This files is also located in PcapPlusPlus root directory. Please do not remove this file" %}

__Setup__ - takes the following parameters:

| `-g`, `--huge-pages` `AMOUNT` | The amount of huge pages to allocate. By default each huge-page size is 2048KB |
| `-i`, `--interface` `NIC_NAME [NIC_NAME ...]` | A space-separated list of all NICs that should move from Linux to DPDK control. Only these NICs can be used by DPDK, the others will stay under Linux control |
| `-m`, `--dpdk-module` `{igb_uio,uio_pci_generic}` | The DPDK module to install. If not specified the default is `igb_uio` |
| `-k`, `--load-kni` | Install the KNI kernel module (not loaded by default) |
| `-p`, `--kni-params` `KNI_PARAMS` | Optional parameters for installing the KNI kernel module |
| `-v`, `--verbose` | Print more verbose output |

If everything goes well the system should be ready to run a DPDK applications and the output should look something like this:

```shell
pcapplusplus@ubunu:~/PcapPlusPlus$ sudo python setup_dpdk.py setup -g 512 -i enp0s3
[INFO] set up hugepages to 512
[INFO] loaded kernel module 'uio'
[INFO] loaded DPDK kernel module 'igb_uio'
[INFO] set interface 'enp0s3' down
[INFO] bound interface 'enp0s3' ['0000:00:03.0'] to 'igb_uio'
[INFO] SETUP COMPLETE
```

__Status__ - shows the interfaces under DPDK and Linux control. In the example below one interface is under DPDK control and the other (`enp0s8`) is under Linux control:

```shell
pcapplusplus@ubunu:~/PcapPlusPlus$ sudo python setup_dpdk.py status

Network devices using DPDK-compatible driver
============================================
0000:00:03.0 '82540EM Gigabit Ethernet Controller 100e' drv=igb_uio unused=e1000,vfio-pci,uio_pci_generic

Network devices using kernel driver
===================================
0000:00:08.0 '82540EM Gigabit Ethernet Controller 100e' if=enp0s8 drv=e1000 unused=igb_uio,vfio-pci,uio_pci_generic *Active*

No 'Baseband' devices detected
==============================

No 'Crypto' devices detected
============================

No 'Eventdev' devices detected
==============================

No 'Mempool' devices detected
=============================

No 'Compress' devices detected
==============================

No 'Misc (rawdev)' devices detected
===================================
```

This command take the following parameters:

| `-v`, `--verbose` | Print more verbose output |

__Restore__ - restores the Linux configuration to its previous state before `setup_dpdk.py setup` was run. Please note that this command uses the `setup_dpdk_settings.dat` file to identify the previous state. If this file was deleted or moved the restore process will most likely fail. Please also note that a machine restart will probably restore most of this configuration, but this command enables restore without a machine restart.

If everything goes well the output should look something like this:

```shell
pcapplusplus@ubunu:~/PcapPlusPlus$ sudo python setup_dpdk.py restore
[INFO] removed hugepages
[INFO] bound device '0000:00:03.0' back to 'e1000'
[INFO] restored interface 'enp0s3'
[INFO] removed 'igb_uio' kernel module
[INFO] RESTORE COMPLETE
```

This command take the following parameters:

| `-v`, `--verbose` | Print more verbose output |


### Initialization at application startup

When using DPDK in your application it should be initialized on application startup. This configuration includes:

- Verify that huge-pages, kernel module(s) and NICs are all set
- Initialize DPDK internal structures and memory, poll-mode drivers etc.
- Prepare CPU cores that will be used by the application
- Initialize packet memory pool
- Configure each NIC controlled by DPDK

These steps are wrapped in one static method that should be called once in the application startup:

```cpp
DpdkDeviceList::initDpdk();
```

If this methods succeeds DPDK is ready to be used in your application.

## Next steps

If you're curious to learn more, please refer to the following resources:

- [DPDK tutorial]({{ site.baseurl }}/docs/tutorials/dpdk)
- DPDK example applications: [DpdkExample-FilterTraffic]({{ site.baseurl }}/docs/examples#dpdkexample-filtertraffic), [DpdkBridge]({{ site.baseurl }}/docs/examples#dpdkbridge), [KniPong]({{ site.baseurl }}/docs/examples#knipong)
- DPDK API reference. A good starting points would be [`DpdkDevice.h` file description]({{ site.baseurl }}/api-docs/_dpdk_device_8h.html#details) and [`DpdkDevice` class description]({{ site.baseurl }}/api-docs/classpcpp_1_1_dpdk_device.html#details)
- You can also find all the unit-tests in the [`Pcap++Test`](https://github.com/seladb/PcapPlusPlus/blob/master/Tests/Pcap%2B%2BTest/main.cpp) project (search for tests that contain "dpdk" or "kni")