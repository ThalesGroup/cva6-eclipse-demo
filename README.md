
# Introduction

In this repository we show how easy it is to reproduce a full debugging environment on the CVA6 processor on the Genesys 2 board. We provide a guide to regenerate the hardware bitsream and how to flash it on the board. Also, this guide explains how to generate the software tools and the compatible Linux image based on Yocto.
Baremetal and Linux application debugging is also demonstrated in command line interface CLI or with the Eclipse graphical interface.

The hardware generate image is based on the commit [cva6:#018dbc4](https://github.com/openhwgroup/cva6/tree/018dbc4210ca5706c41e246c743226811e12b0d8) and the Linux image is based on the commit [meta-cva6-yocto:#b10911d](https://github.com/openhwgroup/cva6-sdk/tree/b10911d115aa7d98d07ec6f6d479d13a13af0975).

# Choose the CVA6 configuration you want

The CVA6 provides multiple configurations. They can be found in [cva6/core/include](cva6/core/include). You can also define your own configuration with the [cva6/config_pkg_generator.py](cva6/config_pkg_generator.py) tool. This python tool will create a file /core/include/gen32_config_pkg.sv or /core/include/gen64_config_pkg.sv depending on the configuration your modifications are based on.

The core configuration will have impacts on the software configuration. Today, Yocto generates Linux images for rv64imafdc, rv32ima or rv32imac. MMU is required for Linux execution (eg. sv0 will not work). The baremetal demo can manage all configurations.

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

If you want to perform the baremetal demo, you will need the full fledge RISC-V toolchain. To compile it, go to the riscv-gnu-toolchain directory, choose a proper install path, the configured arch and abi and run the following command:
```bash
./configure --prefix=RISCV_TOOLCHAIN_INSTALL_PATH --with-cmodel=medany --with-arch=rv32ima --with-abi=ilp32
make
```

** Warning: You will need GCC13 if you want the Bitmanip extension **

### OpenOCD for RISC-V

If you want to use a CLI debug, you will need openocd. Here are the steps to generate it :

```
git clone https://github.com/riscv/riscv-openocd
cd riscv-openocd
./configure --prefix=./install && make && make install
```

you should find the binary in riscv-openocd/install/bin/

# Build Hardware

To build the bitstream, retrieve the CVA6 submodule containing the COREV-APU SoC. You need to specify the correct chosen configuration package, then run the following commands:

```bash
cd cva6
git submodule update --init --recursive
target=cv32a6_imac_sv32 make fpga
```

This step is quite long. You should now have the cva6/corev_apu/fpga/work-fpga/ariane_xilinx.mcs memory configuration file. 

# Flash the Genesys bitstream

be sure that the JP5 jumper is in QSPI position. The internal memory of the Genesys 2 can be configured by following this [guide](https://github.com/openhwgroup/cva6#programming-the-memory-configuration-file). The internal QSPI of the Genesys 2 should now be written with the COREV-APU SoC. 

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

## Build

Yocto is provided by the [meta-cva6-yocto repository](https://github.com/openhwgroup/meta-cva6-yocto). To install it, you need to install its requirements :

```
sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev python3-subunit mesa-common-dev zstd liblz4-tool file locales
```

From the README of meta-cva6-yocto : 

First install the repo tool

```
mkdir ${HOME}/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ${HOME}/bin/repo
chmod a+x ${HOME}/bin/repo
PATH=${PATH}:~/bin
```

Create workspace

```
mkdir cva6-yocto && cd cva6-yocto
repo init -u https://github.com/openhwgroup/meta-cva6-yocto -b main -m tools/manifests/cva6-yocto.xml
repo sync
repo start work --all
```

Setup Build Environment

```
. ./meta-cva6-yocto/setup.sh
```

To build the linux image:
```
MACHINE=cv32a6-genesys2 bitbake core-image-minimal
```

You now should have an image at build/tmp-glibc/deploy/images/${MACHINE}/core-image-minimal-cv32a6-genesys2.wic.gz

## Flash

You then need to flash your SD card with the cva6-sdk target. Insert the SD-card in your host and identify the correct device path with ```dmesg | tail``` command. 
**Warning, providing a bad device path can mess up your system. Please double check**
```bash
gunzip -c build/tmp-glibc/deploy/images/cv32a6-genesys2/core-image-minimal-cv32a6-genesys2.wic.gz | sudo dd of=/dev/sd$ bs=1M iflag=fullblock oflag=direct conv=fsync status=progress
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
