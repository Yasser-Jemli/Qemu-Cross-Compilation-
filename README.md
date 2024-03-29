<h1 align="center">Qemu_Cross_Compilation</h1>


Testing Cross Compiling with QEMU
Having to frequently set up cross-compilers, I figured it would be interesting to write out my current understanding and see what gaps I have. I found it educational to set up some very basic tests with QEMU to see if things actually work like I’d expect.

I’m by no means an expert here, so I’m sure I’m going to show my ignorance a bit, since this covers a pretty large range of topics that I only have peripheral knowledge of.

<h1 align="left">What is Cross Compiling?</h1>

For compiled languages like C/C++ the compiler takes the source code and translates it into processor instructions. Typically, the compiled software is meant to run on machines similar to the one that did the compiling. Targeting a different sort of system is cross compiling.

A lot of my initial confusion came from the wide variety of cross compiling scenarios that occur.

Compiling for a different processor architecture (x86 machine targeting Arm)
Compiling for a different OS (Windows compiling Linux code)
Compiling for a different set of core libraries (glibc and the ABI come up in Linux)
There’s some nuance here since some instruction sets are backwards compatible (x64 and x86), and some operating systems can provide some levels of emulation. Typically, when cross compiling you trade off portability with performance.

Basically it comes down to whether the OS can correctly interpret the layout of the executable, and whether the machine instructions are compatible with the processor.

I found https://elinux.org/images/1/15/Anatomy_of_Cross-Compilation_Toolchains.pdf to be a good primer of more of the details of what Linux tools are included in a cross compiler.

<h1 align="left">Choosing a Cross Compiler </h1>

Choosing a cross compiler mostly comes down to understanding the system where you want to run your binary.

Are there well supported application specific toolchains? For instance Android Studio, or the Arduino IDE. Since these are already tuned for the common used cases, they are a lot easier to set up then more flexible toolchains.
For a more generic cross compiler like GCC, you need to make sure the tool supports the processor and OS you’re targetting. GCC has a naming convention that captures this. From https://web.eecs.umich.edu/~prabal/teaching/resources/eecs373/toolchain-notes.pdf:
Unix cross compiler naming conventions can seem mystifying. If you search for an ARM compiler, you might stumble across the following toolchains: arm-none-linux-gnueabi, arm-none-eabi, arm-eabi, and arm-softfloat-linux-gnu, among others. This might leave you wondering about the method to the naming madness. Unix cross compilers are loosely named using a convention of the form arch[-vendor][-os]-abi. The arch refers to the target architecture, which in our case is ARM. The vendor nominally refers to the toolchain supplier. The os refers to the target operating system, if any, and is used to decide which libraries (e.g. newlib, glibc, crt0, etc.) to link and which syscall conventions to employ. The abi specifies which application binary interface convention is being employed, which ensures that binaries generated by different tools can interoperate.

Are you trying to build a single binary, or a whole system? Tools like Buildroot and Yacto will build an entire system image including the OS and common utilities for a cross compilation target.
<h1 align="center">Walkthrough of Testing RaspberryPi Cross Compiling with QEMU</h1>
<h2 align="left">Picking the Compilers</h2>
<h4 align="left">Building Binaries</h4>
I recently needed to test a code base to see if there were issues cross compiling for the RaspberryPi.

There are many versions of the RaspberryPi that use different processor. In addition you can run many different OS’s. So my first test was to narrow down what my goal was.

Looking at a bunch of information on the different RaspberryPi models https://gist.github.com/fm4dd/c663217935dc17f0fc73c9c81b0aa845, I decided it would be reasonable to target the ARMv7 and ARMv8 instruction sets. Newer boards are backwards compatible, but there’s some extra complication if they’re running a 64bit OS. For the OS I decided to target the 32 and 64 bit Raspien OS.

I decided to go ahead and use GCC as my cross compiler. Cross compilers can be a pain to install since there’s sometimes ambiguity if a build tool is using the headers and compilers from the native system or the cross compiler. To get around this I used a docker image:


FROM debian:stretch-slim
RUN apt-get update \
&& apt-get install -y \
build-essential \
wget \
g++-arm-linux-gnueabihf \
g++-aarch64-linux-gnu \
&& rm -rf /var/lib/apt/lists/*
This installs the 32 and 64 bit arm C++ compilers.

To build the container I would run:

docker build -t cross-test .

and to run it:

docker run -v $(pwd):/mnt/workspace -it cross-test /bin/bash

Once in the container I could build some test applications using the different cpus and instruction sets:


aarch64-linux-gnu-g++ -mcpu=cortex-a53 -mabi=lp64 -mcmodel=tiny -o hello64 hello.cpp 
arm-linux-gnueabihf-g++ -mcpu=cortex-a53 -mfpu=neon-fp-armv8 -mneon-for-64bits -o hello32 hello.cpp 
I looked at:

https://gist.github.com/fm4dd/c663217935dc17f0fc73c9c81b0aa845
https://github.com/abhiTronix/raspberry-pi-cross-compilers
https://www.raspberrypi.org/forums/viewtopic.php?t=11629
https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html
For additional reference on which flags to use.

Understanding the Software Backwards Compatibility
I can inspect the generated binaries with the file and readelf commands:


$ file data/hello32 
data/hello32: ELF 32-bit LSB shared object, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0, BuildID[sha1]=4cf11af4932f12e469f1933e88ecadc0856e62b1, not stripped

$ file readelf --notes data/hello32 

Displaying notes found in: .note.ABI-tag
  Owner                Data size        Description
  GNU                  0x00000010       NT_GNU_ABI_TAG (ABI version tag)
    OS: Linux, ABI: 3.2.0

$ objdump -p data/hello32 

data/hello32:     file format elf32-little

...

Dynamic Section:
  NEEDED               libstdc++.so.6
  NEEDED               libm.so.6
  NEEDED               libgcc_s.so.1
  NEEDED               libc.so.6
  ...

Version References:
  required from libgcc_s.so.1:
    0x0b792655 0x00 05 GCC_3.5
  required from libc.so.6:
    0x0d696914 0x00 04 GLIBC_2.4
  required from libstdc++.so.6:
    0x08922974 0x00 03 GLIBCXX_3.4
    0x0849afa3 0x00 02 CXXABI_ARM_1.3.3

My understanding here is that this binary should be compatible with a version of Linux using library version >= then those printed above.

Looking at https://en.wikipedia.org/wiki/Raspberry_Pi_OS, it looks like these should be compatible the Raspian releases 2016 and later.

For a more complicated program I would need to cross compile all the libraries I was using since I couldn’t use native libraries installed on the build environment.

Hardware Compatibility
When it comes to the hardware, I looked at the table in https://en.wikipedia.org/wiki/Raspberry_Pi#Specifications

Basically all the models I’m interested either support ARMv7-A or ARMv8-A along with the VFPv4 + NEON floating point unit.

Running QEMU
QEMU is an emulator that lets you simulate running an entire other computer in Linux. You need to specify the hardware as well as the software images you’re going to run with.

I was able to pretty closely follow https://github.com/dhruvvyas90/qemu-rpi-kernel/tree/master/native-emulation.

I downloaded a kernel and device tree from https://github.com/dhruvvyas90/qemu-rpi-kernel/tree/master/native-emulation, along with an Raspien image from http://www.cs.tohoku-gakuin.ac.jp/pub/Linux/RaspBerryPi/.

I put these in a data directory and ran QEMU with the command:



qemu-system-aarch64 \
  -m 1024 \
  -M raspi3 \
  -kernel data/kernel8.img \
  -dtb data/bcm2710-rpi-3-b.dtb \
  -sd data/2020-05-27-raspios-buster-lite-armhf.img \
  -append "console=ttyAMA0 root=/dev/mmcblk0p2 rw rootwait rootfstype=ext4" \
  -nographic
Since it doesn’t appear that networking was easily supported, I needed a way to load the binaries I wanted to test onto the emulator.

To do this I would mount the img file on my host machine and copy over the binary. I did this based on the instructions I found here https://azeria-labs.com/emulate-raspberry-pi-with-qemu/


sudo mkdir /mnt/raspbian
sudo mount -v -o offset=272629760 -t ext4 data/2019-09-26-raspbian-buster-lite.img /mnt/raspbian
# hello32 is the test binary compiled earlier
cp data/hello32 /mnt/raspbian/home/pi
sudo umount /mnt/raspbian
Unfortunately, you need to shutdown and restart QEMU to get it to read in the modified files. At least this let me confirm that I was indeed able to run with my cross compiled binaries.

Next I wanted to see if compiling targeting an older arm version would work.

Compiled with: arm-linux-gnueabihf-gcc -march=armv7-a -mfpu=neon-vfpv4 -o hello32_old hello.c

Then ran the 32 bit OS on the same emulated hardware


qemu-system-aarch64 \
  -m 1024 \
  -M raspi3 \
  -kernel data/kernel8.img \
  -dtb data/bcm2710-rpi-3-b.dtb \
  -sd data/2020-05-27-raspios-buster-lite-armhf.img \
  -append "console=ttyAMA0 root=/dev/mmcblk0p2 rw rootwait rootfstype=ext4" \
  -nographic
This also ran with no problem, showing that the instruction set was backwards compatible.

https://www.robopenguins.com/cross-compiling/
