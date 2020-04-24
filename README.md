## Compiling for the local system, on the local system.

Compiling a kernel 101:

* Download kernel source
* tar xfvJ kernel-x.y.z.tar.xz
* cd kernel-x.y.z
* make clean mrproper
* cp <config> .config # If you have one!
* make oldconfig
* make menuconfig
* make -j<number of threads> deb-pkg LOCALVERSION=-mynewkernelname

## Cross-compiling example for a kernel: ARM v7 to i686.

Warning: Cross-compiling isn't straightforward.

Key details
- Date:   2020/04/13
- Build:  Raspberry Pi 4 (ARMv7 2019) 
- Host:   Raspberry Pi 4 (ARMv7 2019) 
- Target: VIA EPIA SN10000EG (2008 i686)
- Kernel: 5.6.3
- GCC:	  8.3.0

Building on the EPIA takes ~2 hours. 
Building on the Pi 4 takes ~20 mins.

Kernel configs have options for:
* Basic motherboard needs (Bus, Storage, Network, USB, Sound, etc)
* Obligatory junk like systemd
* Running Intel PowerTop for power efficiency

#### Build crosstool-ng
```
$ ./config --prefix=/home/pi/ct-ng-source
$ make 
$ make install
$ export PATH="${PATH}:/home/pi/ct-ng-source/bin"
```

#### Building an i686 toolchain with crosstool-ng
```
$ mkdir /home/pi/ct-ng-build
$ cd /home/pi/ct-ng-build
$ touch .config
$ cat >> .config <<EOF
# This is a build for a vendor-less i686 kernel
CT_CONFIG_VERSION="3"
CT_LOCAL_TARBALLS_DIR="/home/pi/ct-ng-build/src"
CT_WORK_DIR="/home/pi/ct-ng-build/.build"

# Target - i686 in this case
CT_ARCH_X86=y
CT_OMIT_TARGET_VENDOR=y
# For 32-bit, i686
CT_ARCH_ARCH="i686"
# For 64-bit, x86_64
# CT_ARCH_64=y

# Operating System - build for Linux
CT_KERNEL_LINUX=y

# Binary Utilities - Linker stuff
CT_BINUTILS_LINKER_LD_GOLD=y 
CT_BINUTILS_GOLD_THREADS=y 
CT_BINUTILS_LD_WRAPPER=y
CT_BINUTILS_PLUGINS=y

# C Compiler - Throw in G++
CT_CC_LANG_CXX=y
EOF
$ ct-ng menuconfig
$ ct-ng build
```

#### Prepare the kernel
```
$ tar xfJ linux-<ver>.tar.xz
$ cd linux
$ cp ../config-x.y.z .config
... apply patches ...
```

#### Configure and build the kernel
```
$ export PATH=$PATH:/home/pi/x-tools/i686-linux-gnu/bin
$ make ARCH=x86 CROSS_COMPILE=i686-linux-gnu- oldconfig
$ make ARCH=x86 CROSS_COMPILE=i686-linux-gnu- menuconfig
... make changes ..
$ make ARCH=x86 CROSS_COMPILE=i686-linux-gnu- -j4 deb-pkg
```

#### Further steps on the target
```
$ sudo dpkg -i linux-image-<ver>_<arch>.deb
$ sudo dpkg -i linux-headers-<ver>_<arch>.deb
$ sudo dpkg -i linux-libc-dev_<ver>_<arch>.deb

#### For building external kernel modules (modpost, fixdep)
$ cd /usr/src/linux-headers-<ver>
$ make M=scripts/mod
$ make M=scripts/basic
```
