
# Introduction

In this repository we show how easy it is to reproduce a full debugging environment on the CVA6 processor on the Genesys 2 board. We provide a guide to regenerate the hardware bitsream and how to flash it on the board. Also, this guide explains how to generate the software tools and the compatible Linux image based on Yocto.
Baremetal and Linux application debugging is also demonstrated in command line interface CLI or with the Eclipse graphical interface.

The hardware generate image is based on the commit [cva6:#018dbc4](https://github.com/openhwgroup/cva6/tree/018dbc4210ca5706c41e246c743226811e12b0d8) and the Linux image is based on the commit [meta-cva6-yocto:#b10911d](https://github.com/openhwgroup/meta-cva6-yocto/tree/b10911d115aa7d98d07ec6f6d479d13a13af0975).

You do not have to update all the git submodules, the README indicate which are required for each step.

# Choose the CVA6 configuration you want

The CVA6 provides multiple configurations. They can be found in [cva6/core/include](cva6/core/include). You can also define your own configuration with the [cva6/config_pkg_generator.py](cva6/config_pkg_generator.py) tool. This python tool will create a file [/core/include/gen32_config_pkg.sv](/core/include/) or [/core/include/gen64_config_pkg.sv](/core/include/) depending on the configuration your modifications are based on.

The core configuration will have impacts on the software and some hardware configurations are not compatible with Linux. Today, Yocto generates Linux images for rv64imafdc or rv32ima. MMU is required for Linux execution (eg. sv0 will not work). The baremetal demo can manage all configurations.

# Requirements

## Hardware requirements

To reproduce this demonstration you will need :

| Reference	                 | URL                                                                             |	Remark                            |
| :------------------------- | :------------------------------------------------------------------------------ | :-------------------------------- |
| Genesys 2 Kintex-7 FPGA Development Board	 | https://digilent.com/shop/genesys-2-kintex-7-fpga-development-board/ |  |
| micro SD Card         |	               | at least 4Gb       |
| 2 micro-USB cables    |	               |       |
| Ethernet cable        |                  | required for Linux application debugging       |

You will need to connect both the UART and the JTAG micro USB of the Genesys 2 to your development machine.

## Bitstream

You will need Vivado 2020.1 to generate the hardware bitstream. Vivado is also needed to program the bitsream onto the Genesys 2 board. 

### Build the bitstream

To build the bitstream, retrieve the CVA6 submodule containing the COREV-APU SoC. You need to specify the correct chosen configuration package. For the rv32ima configuration, run the following commands:

```bash
cd cva6
git submodule update --init --recursive ./
target=cv32a6_ima_sv32_fpga make fpga
```

This step is quite long. You should now have the cva6/corev_apu/fpga/work-fpga/ariane_xilinx.mcs memory configuration file. 

### Flash the Genesys bitstream

be sure that the JP5 jumper is in QSPI position. The internal memory of the Genesys 2 can be configured by following this [guide](https://github.com/openhwgroup/cva6#programming-the-memory-configuration-file). The internal SPI flash of the Genesys 2 should now be written with the COREV-APU SoC. The FPGA is programmed with the content of this flash after power-up. 

## CLI requirement

### RISC-V Toolchain

If you want to perform the CLI debug demo, you will need the full fledge RISC-V toolchain. To compile it, go to the riscv-gnu-toolchain directory, choose a proper install path, the configured arch and abi and run the following command:
```bash
# For the 32 bit configuration without floating registers, adapt accordingly to your configuration
export ARCH=rv32ima
export ABI=ilp32
export RISCV=RISCV_TOOLCHAIN_INSTALL_PATH
# For the 64 bits configuration
#export ARCH=rv64gc 
#export ABI=lp64d
git submodule update --init --recursive riscv-gnu-toolchain
cd riscv-gnu-toolchain
./configure --prefix=$RISCV --with-cmodel=medany --with-arch=$ARCH --with-abi=$ABI

# if you intend to run the baremetal debug 
make
# if you intend to run the linux debug 
# make linux
```

**Warning: You will need GCC13 if you want the Bitmanip extension**

### Sample program

To compile a sample program for baremetal debug, you will need to set the PATH environment variable with the RISC-V toolchain [you generated previously](#risc-v-toolchain).

```bash
PATH=${RISCV}/bin:$PATH
cd cva6-baremetal-bsp
git submodule update --init --recursive ./
cd app
make helloworld
```

For a sample program for Linux debugging, a simple compilation will suffice:
```bash
${RISCV}/bin/riscv[32/64]-unknown-linux-gnu-gcc -g -o helloworld_printf app/helloworld_printf/helloworld_printf.c
```

Other sample programs are available in the app folder. Of course you can run your own program compiled for the CVA6.

**Warning, some errors have been found with the cva6-baremetal-bsp in 64 bits**

## Linux requirement

### Build yocto

Yocto is provided by the [meta-cva6-yocto repository](https://github.com/openhwgroup/meta-cva6-yocto). To install it, you need to install its requirements :

```
sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev python3-subunit mesa-common-dev zstd liblz4-tool file locales
```

From the README of meta-cva6-yocto : 

First install the repo tool

```
curl https://storage.googleapis.com/git-repo-downloads/repo > ${RISCV}/bin/repo
chmod a+x ${RISCV}/bin/repo
PATH=${RISCV}/bin:${PATH}
```

Create workspace

```
mkdir cva6-yocto && cd cva6-yocto
repo init -u https://github.com/openhwgroup/meta-cva6-yocto -b b10911d115aa7d98d07ec6f6d479d13a13af0975 -m tools/manifests/cva6-yocto.xml
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

### Flash yocto

You then need to flash your SD card with the cva6-sdk target. Insert the SD-card in your host and identify the correct device path with ```dmesg | tail``` command. 

**Warning, providing a bad device path can mess up your system. Please double check**
```bash
gunzip -c build/tmp-glibc/deploy/images/cv32a6-genesys2/core-image-minimal-cv32a6-genesys2.wic.gz | sudo dd of=/dev/sd$ bs=1M iflag=fullblock oflag=direct conv=fsync status=progress
```

The SD card should now be formated and written, ready to boot. Insert it in the Genesys2 board and power it on. The console on the UART output should display the linux boot process and log in the system.

## Baremetal CLI requirements 

### OpenOCD for RISC-V

If you want to test the baremetal CLI debug, you will need openocd. Eclipse embedd its own OpenOCD. Here are the steps to generate it :

```
git clone https://github.com/riscv/riscv-openocd
cd riscv-openocd
./bootstrap
./configure --prefix=`realpath ./install`
make
make install
```

you should find the binary in riscv-openocd/install/bin/

## Eclipse requirements

If you want to use the Eclipse graphical environment, you will need to download and install eclipse for embedded developers: 
https://www.eclipse.org/downloads/packages/release/2022-06/r/eclipse-ide-embedded-cc-developers

# Baremetal debugging

Follow these steps only if you are interested in baremetal debugging (no operating system on the target).
Once the CVA6 is flashed in the internal memory of the board, you can debug the core with the JTAG interface. The baremetal application is small and stand-alone, so it can easily be loaded through the JTAG link.

## CLI debugging

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

## Eclipse debugging

WIP

Make sure that the linker option contains "--specs=nosys.specs".

# Linux application debugging

Linux applications cannot be debugged with the OpenOCD tool because it has no knowledge of the Linux internal structures. For linux application debugging we use the gdbserver tool to remotely debug a program.
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

In order to transfer the elf binary, you can launch the SSH server. This step is mandatory for the Eclipse debuging. During the first launch, dropbear will generate a rsa key which take some times.
```bash
# On the Genesys2
/etc/init.d/dropbear start
```

## CLI debugging

Once the connection is set up, you can transfer a program. Let's try to transfer the helloworld_print.riscv previously compiled. 
The debugging can now beggin. Start a gdb server on the CVA6 with:
```bash
# On the Genesys2
gdbserver localhost:3333 helloworld_printf.riscv
```
Then connect to it from your host.
```bash
# On the host
$RISCV/bin/riscv[32/64]-unknown-linux-gnu-gdb helloworld_printf.riscv
```
```gdb
# in gdb
target remote CVA6_IP:3333
c
```

## Eclipse debugging

WIP

Both the Baremetal and the Linux debugging shown previously can be achieved through the Eclipse graphical environment. This chapter explains how to set up the Eclipse projects for remote debugging.

##  Acknowledgement
This activity has received funding from the Key Digital Technologies Joint Undertaking (KDT JU) under grant agreement No 877056. The JU receives support from the European Union’s Horizon 2020 research and innovation programme and Spain, Italy, Austria, Germany, Finland, Switzerland.

![FRACTAL Logo](https://cloud.hipert.unimore.it/apps/files_sharing/publicpreview/jHmgbEb2QJoe8WY?x=1912&y=617&a=true&file=fractal_logo_2.png&scalingup=0)

![EU Logo](https://cloud.hipert.unimore.it/apps/files_sharing/publicpreview/pessWNfeqBfYi3o?x=1912&y=617&a=true&file=eu_logo.png&scalingup=0)
![KDT Logo](https://cloud.hipert.unimore.it/apps/files_sharing/publicpreview/yd7FgKisNgtLPTy?x=1912&y=617&a=true&file=kdt_logo.png&scalingup=0)   
