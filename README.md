
# Introduction

In this repository we show how easy it is to reproduce a full Eclipse debugging environment on Linux running on the CVA6 processor on the GenesysII board. Baremetal and Linux application debugging is demonstrated. We provide harward bitsream and Linux image for those who wants to start faster on the board. We also provide a guide to regenerate them.

Provided images are based on **#cva6_commit_nb** and **#cva6-sdk_commit_nb**.

# Depedencies

## Hardware requirements

To reproduce this demonstration you will need :

| Reference	                 | URL                                                                             |	Remark                            |
| :------------------------- | :------------------------------------------------------------------------------ | :-------------------------------- |
| Genesys 2 Kintex-7 FPGA Development Board	 | https://digilent.com/shop/genesys-2-kintex-7-fpga-development-board/ |  |
| micro SD Card         |	               | at least 4Gb       |
| 2 micro-USB cables    |	               |       |
| Ethernet cable        |                  | required for Linux application debugging       |

You will need to connect both the UART and the FPGA micro USB connectors to your development machine.

## Software dependencies

### Vivado 2020.1

You will need Vivado 2020.1 to generate the hardware bitstream. Vivado is also needed to program the bitsream onto the GenesysII board. ~~After installation, if you want to regenerate the bitstream, the GenesysII board's configuration files need to be installed. You can find them here :~~

### Buildroot

Buildroot is provided by the [cva6-sdk repository](https://github.com/openhwgroup/cva6-sdk). To install it, simply clone it, install its requirements and run ```make all```. 

### Debugging

To debug your software, you can use eclipse IDE or the common line interface of gdb. In both case, you will need the tools provided by the buildroot toolchain.

#### Eclipse

You will need to download and install eclipse for embedded developers: 
https://www.eclipse.org/downloads/packages/release/2022-06/r/eclipse-ide-embedded-cc-developers

# Build Hardware

To build the bitstream

# Baremetal debugging

# Build Linux image

# Linux application debugging

## Eclipse's project creation

then you can create a new project
remove no spec argument
Specify the proper toolchain path
create a new launch configuration
configure ssh
launch debug 
