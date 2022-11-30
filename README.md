
# Introduction

In this repository we show how easy it is to reproduce a full debugging environment on the CVA6 processor on the Genesys 2 board. Baremetal and Linux application debugging is demonstrated in command line interface CLI or with the Eclipse graphical interface. We provide hardware bitsream and Linux image for those who wants to start faster on the board. We also provide a guide to regenerate them.

The hardware provided image is based on the commit [cva6:#d209b04](https://github.com/openhwgroup/cva6/tree/d209b0406dba696744185001bf7257af7d1d8890) and the Linux image is based on the commit [cva6-sdk:#cb35d1d](https://github.com/openhwgroup/cva6-sdk/tree/cb35d1dba01da51b1489fb109f1b5598bd655267).

# Required depedencies

## Hardware requirements

To reproduce this demonstration you will need :

| Reference	                 | URL                                                                             |	Remark                            |
| :------------------------- | :------------------------------------------------------------------------------ | :-------------------------------- |
| Genesys 2 Kintex-7 FPGA Development Board	 | https://digilent.com/shop/genesys-2-kintex-7-fpga-development-board/ |  |
| micro SD Card         |	               | at least 4Gb       |
| 2 micro-USB cables    |	               |       |
| Ethernet cable        |                  | required for Linux application debugging       |

You will need to connect both the UART and the JTAG micro USB of the Genesys 2 to your development machine.

## Software requirements

### Vivado 2020.1

You will need Vivado 2020.1 to generate the hardware bitstream. Vivado is also needed to program the bitsream onto the Genesys 2 board. ~~After installation, if you want to regenerate the bitstream, the GenesysII board's configuration files need to be installed. You can find them here :~~

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

# Build Hardware

If you want to re-build the bitstream, retrieve the CVA6 submodule containing the COREV-APU SoC. Then run the following script:

```
cd cva6
git submodule update --init --recursive
make fpga
```

This step is quite long. You should now have the cva6/corev_apu/fpga/work-fpga/ariane_xilinx.mcs memory configuration file. 

# Flash the Genesys 2 QSPI

Be sure that the JP5 jumper is in QSPI position. The internal memory of the Genesys 2 can be configured by following this [guide](https://github.com/openhwgroup/cva6#programming-the-memory-configuration-file). The internal QSPI of the Genesys 2 should now be written with the COREV-APU SoC. 

# Baremetal debugging

Once the CVA6 is flashed in the internal memory of the board, you can debug the core with the JTAG interface.

## GDB CLI

The Buildroot toolchain provides openocd and gdb for the CVA6 architecture. Compile a sample program with :

```
cat << EOF | riscv64-linux-gcc -march=rv64g  -mabi=lp64d -x c -o hello.elf -
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

You should have a gdb program with control on the CVA6. You can look around and press "c" to continue the execution. The test string is displayed on the console terminal.

# Build Linux image

*TODO*

# Linux application debugging

*TODO*

# Eclipse debugging

*TODO*

Both the Baremetal and the Linux debugging shown previously can be achieved through the Eclipse graphical environment. This chapter explains how to set up the Eclipse projects for remote debugging.

## Installation 

*TODO*
 
If you want to use the Eclipse graphical environment, you will need to download and install eclipse for embedded developers: 
https://www.eclipse.org/downloads/packages/release/2022-06/r/eclipse-ide-embedded-cc-developers

## Eclipse's project creation

then you can create a new project
remove no spec argument
Specify the proper toolchain path
create a new launch configuration
configure ssh
launch debug 
