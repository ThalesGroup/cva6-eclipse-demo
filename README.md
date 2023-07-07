
# Introduction

In this repository we show how easy it is to reproduce a full debugging environment on the CVA6 processor on the Genesys 2 board. We provide a guide to regenerate the hardware bitsream and how to flash it on the board. Also, this guide explains how to generate the software tools and the compatible Linux image based on Yocto.
Baremetal and Linux application debugging is also demonstrated in command line interface CLI or with the Eclipse graphical interface.

The hardware generate image is based on the commit [cva6:#018dbc4](https://github.com/openhwgroup/cva6/tree/018dbc4210ca5706c41e246c743226811e12b0d8) and the Linux image is based on the commit [meta-cva6-yocto:#b10911d](https://github.com/openhwgroup/cva6-sdk/tree/b10911d115aa7d98d07ec6f6d479d13a13af0975).

# Choose the CVA6 configuration you want

The CVA6 provides multiple configurations. They can be found in [cva6/core/include](cva6/core/include). You can also define your own configuration with the [cva6/config_pkg_generator.py](cva6/config_pkg_generator.py) tool.

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

You will need Vivado 2020.1 to generate the hardware bitstream. Vivado is also needed to program the bitsream onto the Genesys 2 board. 


### RISC-V Toolchain

If you want to perform the baremetal demo, you will need the full fledge RISC-V toolchain. To compile it, go to the riscv-gnu-toolchain directory, choose a proper install path and run the following command:
```bash
./configure --prefix=RISCV_TOOLCHAIN_INSTALL_PATH --with-cmodel=medany
make
```

### Buildroot

Buildroot is provided by the [cva6-sdk repository](https://github.com/openhwgroup/cva6-sdk). To install it, you need to install its requirements found in its README and then run the next lines.

```bash
cd cva6-sdk
git submodule update --init --recursive
make all
make -C buildroot host-gdb
make -C buildroot host-openocd
```

You now have a RISC-V toolchain corresponding to the CVA6 architecture. It is located in cva6-sdk/buildroot/output/host/bin/. You must add it to the PATH environment variable of your building terminal:

```bash
PATH=`realpath cva6-sdk/buildroot/output/host/bin/`:$PATH
```

# Build Hardware

To re-build the bitstream, retrieve the CVA6 submodule containing the COREV-APU SoC. Then run the following commands:

```bash
cd cva6
git submodule update --init --recursive
make fpga
```

This step is quite long. You should now have the cva6/corev_apu/fpga/work-fpga/ariane_xilinx.mcs memory configuration file. 

# Flash the Genesys bitstream

Be sure that the JP5 jumper is in QSPI position. The internal memory of the Genesys 2 can be configured by following this [guide](https://github.com/openhwgroup/cva6#programming-the-memory-configuration-file). The internal QSPI of the Genesys 2 should now be written with the COREV-APU SoC. 

# Baremetal debugging

Once the CVA6 is flashed in the internal memory of the board, you can debug the core with the JTAG interface.

## GDB CLI

Compile a sample program with :

```bash
PATH=RISCV_TOOLCHAIN_INSTALL_PATH/bin:$PATH
cd cva6-baremetal-bsp/app
git checkout 64bits
make helloworld
```

Connect a Console on the UART output of the Genesys2 :

```bash
# Port can change depending on the devices plugged on your system
gtkterm -p /dev/ttyUSB0
```
Then take control of the SoC and launch the program :

```bash
openocd -f cva6/corev_apu/fpga/ariane.cfg& 
riscv64-linux-gdb helloworld.riscv -x gdb_command
```

You should have a gdb program with control on the CVA6. You can look around and press "c" to continue the execution. The test string is displayed on the console terminal.

# Build and flash Linux image

To build the linux image, go to cva6-sdk submodule and type 
```bash
cd cva6-sdk
git submodule update --init
make images
```

You now should have a populated cva6-sdk/install64 folder.

You then need to flash your SD card with the cva6-sdk target. Insert the SD-card in your host and identify the correct device path with ```dmesg | tail``` command. 
**Warning, providing a bad device path can mess up your system. Please double check**
```bash
sudo -E make flash-sdcard SDDEVICE=/dev/sd$
```

The SD card should now be formated and written, ready to boot. Insert it in the Genesys2 board and power it on. The console on the UART output should display the linux boot process and log in the system.

# Linux application debugging

In order to debug on Linux, a remote connection is mandatory. As the UART is already used by the console, it is advised to use a ssh connection. You will need to plug the Ethernet from the Genesys2 board to your host computer. In order to connect to the Genesys2, you either have to set up a DHCP server on your host and then call the dhcp client on the Genesys2 with :
```bash
# On the Genesys2
udhpc
```
Or set a static IP on the same network as the host like this:
```bash
# On the Genesys2
ifconfig eth0 192.168.##.##
```

Once the connection is set up, you can transfer a program. Let's try to transfer the hello.elf previously compiled. 
The debugging can now beggin. Start a gdb server on the CVA6 with:
```bash
# On the Genesys2
gdbserver localhost:3333 hello.elf
```
Then connect to it from your host.
```bash
gdb hello.elf
```
```gdb
target remote CVA6_IP:3333
c
```

# Eclipse debugging

Both the Baremetal and the Linux debugging shown previously can be achieved through the Eclipse graphical environment. This chapter explains how to set up the Eclipse projects for remote debugging.

## Installation 

If you want to use the Eclipse graphical environment, you will need to download and install eclipse for embedded developers: 
https://www.eclipse.org/downloads/packages/release/2022-06/r/eclipse-ide-embedded-cc-developers

## Eclipse's project creation

### Baremetal debugging

Make sure that the linker option contains "--specs=nosys.specs".

### Linux debugging

In Eclipse IDE, you can create a new project with File>New>C/C++ Project.

Select "C Managed Build" and click Next.

Let's start with a simple hello world:
In the executable list, select "Hello World RISC-V C Project"
Give your project a name and click Next.

You must remove the linker option, set this box empty and click next twice.

You now must indicate the toolchain. You can use the buildroot one by specifying the ```cva6-sdk/buildroot/output/host/bin``` path in the toolchain path box. Then click finish.

Once the project is created, right click on the project, select "Properties" and go to C/C++Build>Settings>Toolchains tab. Change the prefix to "riscv64-linux-" and make sure the bottom boxes (Create flash image, Create extended listing and Print size) are left un-ticked.

Now you can build the project: right click on it and select "Build project". An elf file should appear in the Debug folder of your project.


create a new launch configuration
configure ssh
launch debug 
