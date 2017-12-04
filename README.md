# Rocket Core on Zynq FPGAs

This page documents the setup process for building Rocket Cores to work on Zynq FPGAs as well as some of the basic architectural components of a Rocket Core. The installation instructions were tested on a bento/ubuntu-16.04 Vagrant box with 2GB RAM and 2 cores, except where otherwise noted. A Vagrantfile with these configurations has been provided.

Last update: 12/4/17

## Introduction

### Terms and Knowledge
Some terms and background knowledge will be essential to understanding this research project. The pre-requisite knowledge will be explained in the following sections.

#### FPGAs and Zynq-7000
An FPGA is a programmable hardware board. It makes prototyping hardware designs fast, cheap, and simple. At Georgia Tech, an undergraduate computer science student's first exposure to FPGA development might be in CS 3220 - Processor Design, where the student is challenged to implement a simple five stage pipelined processor on an FPGA.

In this research project, we worked with a specific brand and model FPGA board. The Zybo Zynq-7000, distributed by Digilent, is a ready-to-use, entry-level FPGA. It contains a dual-core ARM A9 processor and supports several peripheral components such as Ethernet, video, and audio. Note that the Zybo Zynq-7000 has been replaced by the Zybo Z7-10 as of September 21st, 2017. Purchase information and a feature list can be found here:

<a href="http://store.digilentinc.com/zybo-zynq-7000-arm-fpga-soc-trainer-board/" target="_blank">http://store.digilentinc.com/zybo-zynq-7000-arm-fpga-soc-trainer-board/</a>

The base board only comes with the FPGA itself and does not include any cables, SD cards, or installation materials. To make the most out of this board, it is recommended to also purchase:

<ul>
    <li>x1: USB Cable - 2.0 A Male to Micro B</li>
    <li>x1: microSD card with adapter - Zybo supports 8GB safely, unknown if it supports more. According to <a href="https://www.xilinx.com/support/answers/50991.html" target="_blank">here</a>, it supports an unlimited SD card size. However, booting from larger than 32GB seems to require special configuration according to <a href="https://forums.xilinx.com/t5/Zynq-All-Programmable-SoC/Booting-Zynq-from-a-64GB-SD-Card/td-p/555049" target="_blank">this</a>.</li>
    <li>x1: 5V/2.5A power adapter - coax, center-positive 2.1mm internal-diameter plug. This is desirable if running more complex or powerful designs, such as the Rocket Core and RISC-V Linux used in this research project.</li>
</ul>

Here is a link to the official Zybo reference manual for more information on the Zybo's I/O ports and capabilities:<br>
<a href="https://reference.digilentinc.com/_media/zybo:zybo_rm.pdf" target="_blank">Zynq-7000 Zybo Reference Manual</a>

### ISAs and RISC-V
ISA is an acronym


## Chisel Framework

Chisel is an open-source hardware construction language developed at UC Berkeley that supports advanced hardware design using highly parameterized generators. It allows users to design hardware in a higher-level programming language, Scala, instead of directly interfacing with Verilog.

<a href="https://github.com/ucb-bar/chisel-tutorial/wiki" target="_blank">Here</a> is a succinct yet informative Wiki-Book containing information on the most important features of Chisel. Chisel can be set up from scratch on a Unix based system using the following commands. These commands are just a compiled version of all the installation processes found on various tutorials for how to install Java, Scala, and Chisel.

### Installing Chisel
1. Chisel is a hardware construction language built in Scala. Scala requires the JVM to run, so first install the Java 8 SDK.
```
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get -y install oracle-java8-installer
```

2. Install Scala.
```
echo "deb https://dl.bintray.com/sbt/debian /" | sudo tee -a /etc/apt/sources.list.d/sbt.list
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2EE0EA64E40A89B84B2DF73499E82A75642AC823
sudo apt-get update
sudo apt-get install sbt
```

3. Install necessary build tools. Many of these are necessary for installing riscv-tools later, so be sure to install all of them.
```
sudo apt-get install git make g++ autoconf automake autotools-dev curl device-tree-compiler
sudo apt-get install libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo
sudo apt-get install gperf libtool patchutils bc zlib1g-dev pkg-config libusb-1.0-0-dev
```

4. Download Chisel (and RISCV) dependencies.
```
git clone http://git.veripool.org/git/verilator
cd verilator
git pull
git checkout verilator_3_886
unset VERILATOR_ROOT
autoconf
./configure
make
sudo make install
cd ..
```

### Chisel Tutorial
Once you have installed Chisel, you can check that it works properly by downloading the Chisel Tutorial and running a basic test.
Interested users can learn the basics of Chisel (and Scala) in a small tutorial developed by Berkeley here:
https://github.com/ucb-bar/chisel-tutorial

1. Download and Set-Up Chisel Tutorial
```
git clone https://github.com/ucb-bar/chisel-tutorial.git
cd chisel-tutorial
git fetch origin
git checkout release
sbt run
```

2. Test Chisel Tutorial Setup. If everything installed properly, this should print "Tutorials passing: 1".
```
sbt
> test:run-main problems.Launcher Mux2
```

## Rocket Chip Generator
Originally developed by the Berkeley Architecture Research group (UCBAR), the Rocket Chip Generator creates instances of the RISC-V architecture in Verilog. Written using Chisel, the Rocket Chip Generator can be quickly parameterized to build different configurations of RISC-V hardware. The Rocket Chip open-source repository contains all the necessary tools to build and run the chip and can be found here: https://github.com/freechipsproject/rocket-chip.

Here are the steps to download and set-up the Rocket Chip Generator. Before running this, please make sure you have installed all the Scala and Chisel dependencies as shown in the previous section. Without them, running `build.sh` will fail and you will have to re-do these steps. Note that the recursive submodule update and the build script will take a while, so be prepared to grab a coffee in the meantime.

1. Recursively Download Rocket Chip Submodules
```
git clone https://github.com/ucb-bar/rocket-chip.git
cd rocket-chip
git submodule update --init
mkdir riscv-built-toolchain
mv riscv-tools riscv-built-toolchain
cd riscv-built-toolchain
export TOP=$(pwd)
cd riscv-tools
git submodule update --init --recursive
```

2. Set up RISC-V environment variables and dependencies.
```
export RISCV=$TOP/riscvexport
PATH=$PATH:$RISCV/binexport
MAKEFLAGS="$MAKEFLAGS -j2" # Assuming you have 2 cores on your host system
./build.sh
cd $TOP
echo -e '#include <stdio.h>\n int main(void) { printf("Hello world!\\n"); return 0; }' > hello.c
riscv64-unknown-elf-gcc -o hello hello.c
spike pk hello
```

If all went well, your screen should print "Hello world!". For additional debugging help, you may want to consult the riscv-tools GitHub README <a href="https://github.com/riscv/riscv-tools/tree/aca8ec71a3ad9adfc988bdf75306ffe70cbc12e5" target="_blank">here</a>.

## Configuring Rocket Chip Memory
To change the configuration of rocketchip, you first need to change the configuration file and modify/add what configuration you want. The `Configs.scala` file with example configurations can be found at `common/src/main/scala/coreplex/Configs.scala`. There are many parameters that can be figured for both the `icache` and the `dcache` such as `rowBits`, `nSets`, `nWays, `nTLBEntries`, `nMSHRs`, and `blockBytes`. After changing `Configs.scala`. In order to create your own configuration project, a configuration file needs to be added at `common/src/main/scala`. The Verilog for the project can then be regenerated by calling:

```
make rocket CONFIG_PROJECT=ProjectName CONFIG=ConfigName
```

After that, the build process is normal and all that needs to be called are:

```
make project
make fpga-images-zybo/boot.bin
```

This should create the project, generate the bitstream, and create a new `boot.bin` to copy onto the SD card that can then be inserted onto the board.
