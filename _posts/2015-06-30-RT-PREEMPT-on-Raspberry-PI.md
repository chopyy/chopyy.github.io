---
layout: post
title:  "Real-Time capable Linux kernel for the Raspberry PI"
date:   2015-06-30 22:56:07
categories: real-time
---

# Motivation

Embedded systems may need real-time capabilities for controlling, driving actuators, reading out sensor data *in time* with a given rate. This guideline shows how to apply the PREEMPT_RT patch on a Raspberry PI (RPI) v2. This guide includes downloading the sources, cross-compiling the kernel on a PC architecture and finally flashing it on the RPI.

> This guide only covers applying the PREEMPT_RT patch. It does not show how to optimize the kernel configuration to reduce the worst case latency of the real-time kernel. For input on how to optimize your real-time kernel go to [OSADL Real-time optimization](https://www.osadl.org/Real-time-optimization.qa-farm-latency-optimization.0.html).

# Configuration

I'm cross-compiling the kernel on an Intel i7 CPU with 8 GB RAM that runs Ubuntu Linux as my host system, because it will compile a lot faster than compiling the kernel on the RPI itself. As the embedded device I'm using a Raspberry PI v2 Model B.

## Downloading and Patching
There are several possible kernels you can use to build a kernel with PREEMPT_RT. The easiest way to get a real-time kernel running on the RPI is to use the raspberry kernel from their GitHub repository. In this guide I will briefly describe the way for patching and building the real-time kernel with the kernel source from the GitHub repository of Raspberry. I will try to release a guide for patching and running a Vanilla kernel with PREEMPT_RT patch on the RPI soon.

Clone the Raspberry kernel from GitHub:

```bash
cd /usr/src/
mkdir kernels
cd kernels
git clone --depth=1 https://github.com/raspberrypi/linux
```

Look out for the version of the kernel. You will find the information in the Makefile and it should show something like below:

```
VERSION = 3
PATCHLEVEL = 18
SUBLEVEL = 11
```

Thus, you will need the realtime patch 3.18.11:

```bash
cd /usr/src/kernels
sudo wget https://www.kernel.org/pub/linux/kernel/projects/rt/3.18/older/patch-3.18.11-rt7.patch.xz
```

If your kernel version differs from the one described in this guide, search for the appropriate kernel patch on [https://www.kernel.org/pub/linux/kernel/projects/rt/](https://www.kernel.org/pub/linux/kernel/projects/rt/) and download it like described above.

Rename the folder you cloned from GitHub, because we will patch it in the next step:

```bash
cd /usr/src/kernels
sudo mv linux linux-rt
```

Go into your *linux-rt* directory and patch it with the downloaded PREEMPT_RT patch by:

```bash
sudo -i
cd /usr/src/kernels/linux-3.18.11-rt/
sudo xz -dc /usr/src/kernels/patch-3.18.11-rt7.patch.xz | patch -p1
```

**Note:** This step has to be executed as root explicitly (sudo -i). This is obsolete: The recommended way is to patch, configure and build the kernel under the user's home directory.

## Cross Compiler
There is more than one ARM cross compiler for Ubuntu Linux. We will use the provided compiler from the *Raspberry Pi tools section* on GitHub.

```bash
cd /opt/
git clone git://github.com/raspberrypi/tools.git --depth 1
```

**Note:** I renamed the cloned `tools` folder to `rpi-tools`. So my cross-compiler is now located under `/opt/rpi-tools`.

## Configuration
Now comes the exciting part: Configuring the kernel! But before you are able to configure the kernel and enable the fully preemptible kernel, make sure you installed all requirements. Then get the linux kernel ready to compile:

```bash
sudo apt-get install libncurses5-dev
make mrproper
```

Set an environment variable for the prefix of the cross compiler toolchain:

```bash
export CCPREFIX=/opt/rpi-tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf-
```

Make an initial config for the Raspberry build:

```bash
sudo make ARCH=arm CROSS_COMPILE=${CCPREFIX} bcm2709_defconfig
```

In the same terminal where you exported the environment variable run your first make command to start the kernel configuration menu:

```bash
KERNEL=kernel7
sudo make ARCH=arm CROSS_COMPILE=${CCPREFIX} menuconfig
```

This target will start up a graphical interface to configure the kernel.

If you configured the ARM build the Preemption Model of the PREEMPT patch can be found under *Kernel Features* -> **Preemption Model** as shown in the following snippet:

```bash
[ ] Symmetric Multi-Processing
[ ] Architected timer support
    Memory split (3G/1G user/kernel split) --->
[ ] Support for the ARM Power State Coordination Interface (PCSI)
    Preemption Model (Fully Preemptible Kernel (RT)) --->
```

Choose **Fully Preemptible Kernel (RT)** here.

**Note:** If this option is not available the patch was not successful and something went wrong. Try to start from the beginning and patch the kernel again.

## Compiling
You can now compile the kernel on the PC. If you are working on a multi-processor platform enable parallel jobs for a faster build by typing `-j <n>` where <n> is the number of parallel jobs that shall be used. This should be 1.5 times your number of cores. Using this option makes the build about 50% faster on a modern Intel/AMD processor. Now you can imagine how long it would take to compile the kernel directly on the RPI itself (more than 3 hours I guess).

For the raspberry kernel simply execute the following commands:

```bash
cd /usr/src/kernels/linux-rt
sudo make ARCH=arm CROSS_COMPILE=${CCPREFIX} zImage modules dtbs -j5
```

After successful compilation the compiled kernel is located under *arch/arm/boot/*.

## Flash the compiled kernel
We can now flash the image onto the RPI. First, plug in your SD card from RPI and type `lsblk` to list all drives:

This should show something like below:

```bash
NAME
mmcblk0
|-mmcblk0p1
|-mmcblk0p2
```

Install the modules of the compiled kernel. Make sure you're executing the command from your kernel folder:

```bash
sudo make ARCH=arm CROSS_COMPILE=${CCPREFIX} INSTALL_MOD_PATH=/media/name/13d368bf-6dbf-4751-8ba1-88bed06bef77/ modules_install
```

Back-up your old kernel and copy the real-time kernel to the SD card:

```bash
cd /usr/src/kernels/linux-rt
sudo cp /media/name/boot/$KERNEL.img /media/name/boot/$KERNEL-backup.img
sudo scripts/mkknlimg arch/arm/boot/zImage /media/name/boot/$KERNEL-rt.img
sudo cp arch/arm/boot/dts/*.dtb /media/name/boot/
sudo cp arch/arm/boot/dts/overlays/*.dtb* /media/name/boot/overlays/
```

Finally, adjust the config.txt file in `/boot/config.txt` and add the following line:

```bash
kernel=kernel7-rt.img
```

## Running and Testing
After you transferred the kernel onto the RPI, plug in your SD card and boot the controller. Test if the kernel build with the PREEMPT_RT patch runs as expected by typing `uname -a`.

It should show something similiar to this:

```bash
Linux raspberrypi 3.18.14-rt7-v7+ #1 SMP PREEMPT RT Fri Jun 12 12:08:45 CEST 2015 armv7l GNU/Linux
```

There is common toolchain to test the kernel's real-time performance and especially the worst case latency. Log in to your RPI via SSH and clone the following GitHub repository:

```bash
ssh pi@141.7.28.126
git clone git://git.kernel.org/pub/scm/linux/kernel/git/clrkwllms/rt-tests.git
cd rt-tests
make
```

In a terminal execute **cyclictest**:

```bash
cd rt-tests
sudo ./cyclictest -p99 -t -n
```
Where -p defines the priority, -t parametrizes the number of threads and -n is important for using clock_nanosleep().

**Note:** Per default the latencies are in microseconds.

Yet, this measurement isn't meaningful as long as there is no high system load, to get the worst case latency. Open another terminal and start **hackbench**:

```bash
cd rt-tests
sudo ./hackbench -l5000000
```

# Troubleshooting

## Raspberry 2 freezes on execution of cyclictest and hackbench
Due to the execution of **cyclictest** for measuring the kernel's worst case latency and **hackbench** to produce a high system load, it may occur that the RPI freezes. This kind of  behavior is reproducible by using kernel version 3.18.11. This is due to a change of an FIQ handler. I didn't had the time to find out more about this false behavior. Maybe I will add an explanation why the RPI freezes.

Until then, I found a hotfix by adding the following options in front of your `/boot/cmdline.txt`:

```
dwc_otg.fiq_enable=0 dwc_otg.fiq_fsm_enable=0 dwc_otg.nak_holdoff=0
```


# References
* [http://elinux.org/Raspberry_Pi_Kernel_Compilation](http://elinux.org/Raspberry_Pi_Kernel_Compilation)
* [https://www.raspberrypi.org/documentation/linux/kernel/building.md](https://www.raspberrypi.org/documentation/linux/kernel/building.md)
* [https://www.osadl.org/?id=87](https://www.osadl.org/?id=87)

