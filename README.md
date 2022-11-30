
# Introduction

In this repository we show how easy it is to reproduce a full debugging environment on the CVA6 processor on the GenesysII board. Baremetal and Linux application debugging is demonstrated in command line interface CLI or with the Eclipse graphical interface. We provide harward bitsream and Linux image for those who wants to start faster on the board. We also provide a guide to regenerate them.

Provided images are based on **#cva6_commit_nb** and **#cva6-sdk_commit_nb**.

# Required depedencies

## Hardware requirements

To reproduce this demonstration you will need :

| Reference	                 | URL                                                                             |	Remark                            |
| :------------------------- | :------------------------------------------------------------------------------ | :-------------------------------- |
| Genesys 2 Kintex-7 FPGA Development Board	 | https://digilent.com/shop/genesys-2-kintex-7-fpga-development-board/ |  |
| micro SD Card         |	               | at least 4Gb       |
| 2 micro-USB cables    |	               |       |
| Ethernet cable        |                  | required for Linux application debugging       |

You will need to connect both the UART and the FPGA micro USB connectors to your development machine.

## Software requirements

### Vivado 2020.1

You will need Vivado 2020.1 to generate the hardware bitstream. Vivado is also needed to program the bitsream onto the GenesysII board. ~~After installation, if you want to regenerate the bitstream, the GenesysII board's configuration files need to be installed. You can find them here :~~

### Buildroot

Buildroot is provided by the [cva6-sdk repository](https://github.com/openhwgroup/cva6-sdk). To install it, you need to install its requirements found in its README and then run the next lines.

```
cd cva6-sdk
git submodule update --init --recursive
make all
```

You now have a RISC-V toolchain corresponding to the CVA6 architecture. It is located in cva6-sdk/buildroot/output/host/bin/. You must add it to the PATH environment variable of your building terminal:

```
PATH=`realpath cva6-sdk/buildroot/output/host/bin/`:$PATH
```

### Debugging

To debug your software, you can use eclipse IDE or the common line interface of gdb. In both case, you will need the tools provided by the buildroot toolchain.

# Build Hardware

If you want to re-build the bitstream, retrieve the CVA6 submodule containing the COREV-APU SoC. Then run the following script:

```
cd cva6
git submodule update --init --recursive
make fpga
```

This step is quite long. You should now have the cva6/corev_apu/fpga/work-fpga/ariane_xilinx.mcs memory configuration file. 

# Flash the Genesys 2 QSPI

Be sure that the JP5 jumper is in QSPI position. The internal memory of the Genesys 2 can be configured by folowing this [guide](https://github.com/openhwgroup/cva6#programming-the-memory-configuration-file). The internal QSPI of the Genesys 2 should now be written with the COREV-APU SoC. 

# Baremetal debugging

Once the CVA6 is flashed in the internal memory of the board, you can debug the core with the JTAG interface.

## GDB CLI

The Buildroot toolchain provides openocd and gdb for the CVA6 architecture. Compile a sample program with :

```
cat << EOF | riscv64-linux-gcc -march=rv64g  -mabi=lp64 -x c -o hello.elf -
#include <stdio.h>
int main() { printf("Hello World CVA6!\n"); }
EOF
```

Connect a Console on the UART output of the Genesys2 :

```
# Port can change depending on the devices plugged on your system
gtkterm -p /dev/ttyUSB0
```
Then take control of the SoC and launch the program :

```
openocd -f cva6/corev_apu/fpga/ariane.cfg& 
riscv64-linux-gdb hello.elf -x gdb_command
```

You should have 

# Build Linux image

# Linux application debugging

# Eclipse debugging

Both the Baremetal and the Linux debugging shown previously can be achieved through the Eclipse graphical environment. This chapter explains how to set up the Eclipse projects for remote debugging.

## Installation 

If you want to use the Eclipse graphical environment, you will need to download and install eclipse for embedded developers: 
https://www.eclipse.org/downloads/packages/release/2022-06/r/eclipse-ide-embedded-cc-developers

## Eclipse's project creation

then you can create a new project
remove no spec argument
Specify the proper toolchain path
create a new launch configuration
configure ssh
launch debug 
