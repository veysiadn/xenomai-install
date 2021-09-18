# Xenomai real-time patch installation

## Guides

- [Xenomai docs](http://xenomai.org/installing-xenomai-3-x/#Installation_steps)
- [Readthedocs.io](http://rtt-lwr.readthedocs.io/en/latest/rtpc/xenomai.html)
- [Xenomai 3.1](http://rtt-lwr.readthedocs.io/en/latest/rtpc/xenomai3.html)
- [Blog Reds](https://blog.reds.ch/?p=1308)
- [Ethercat Master - Xenomai - SOEM Paper](https://www.koreascience.or.kr/article/JAKO202121137920799.pdf)


## Download links
- [Linux kernel](https://www.kernel.org/pub/linux/kernel/v5.x/)
- [Xenomai](  http://git.xenomai.org/xenomai-3.git)
- [I-pipe](https://xenomai.org/downloads/ipipe/v5.x/x86/)


## Commands

### Prerequisites

```sh
sudo apt-get update
sudo apt install -y qt5-default git curl
sudo apt-get install -y devscripts debhelper dh-kpatches findutils kernel-package libncurses-dev fakeroot zlib1g-dev autotools-dev 
sudo apt-get install -y autoconf automake libtool git libssl-dev libelf-dev libcurses5-dev bison flex

```


### Installation

```sh
#!/bin/sh

echo
echo "## Cloning Xenomai"
mkdir ~/xenomai-build
cd ~/xenomai-build/
git clone https://git.xenomai.org/xenomai-3.git

echo
echo "## Configuring Xenomai"
cd xenomai-3
./scripts/bootstrap
cd ..

echo
echo "Downloading Linux Kernel"
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.4.133.tar.xz
tar -xf linux-5.4.133.tar.xz
wget https://xenomai.org/downloads/ipipe/v5.x/x86/ipipe-core-5.4.133-x86-6.patch

echo
echo "Patching Linux Kernel"
cd linux-5.4.133
./../xenomai-3/scripts/prepare-kernel.sh --arch=x86_64 --ipipe=../ipipe-core-5.4.133-x86-6.patch
cd ..


```

Now configure the kernel
```sh
make menuconfig
```

__Recommended options:__

- ![Image](https://github.com/veysiadn/xenomai-install/blob/master/XenomaiKernelConfig.png)

```
* General setup
  --> Local version - append to kernel release: -xenomai-3.0.5
  --> Timers subsystem
      --> High Resolution Timer Support (Enable)
* Xenomai/cobalt
  --> Sizes and static limits
    --> Number of registry slots (512 --> 4096)
    --> Size of system heap (Kb) (512 --> 4096)
    --> Size of private heap (Kb) (64 --> 256)
    --> Size of shared heap (Kb) (64 --> 256)
    --> Maximum number of POSIX timers per process (128 --> 512)
  --> Drivers
    --> RTnet
        --> RTnet, TCP/IP socket interface (Enable)
            --> Drivers
                --> New intel(R) PRO/1000 PCIe (Enable)
                --> Realtek 8169 (Enable)
                --> Loopback (Enable)
        --> Add-Ons
            --> Real-Time Capturing Support (Enable)
* Pocessor type and features
  --> Enable maximum number of SMP processors and NUMA nodes (Disable)
  // Ref : http://xenomai.org/pipermail/xenomai/2017-September/037718.html
* Power management and ACPI options
  --> CPU Frequency scaling
      --> CPU Frequency scaling (Disable)
  --> ACPI (Advanced Configuration and Power Interface) Support
      --> Processor (Disable)
  --> CPU Idle
      --> CPU idle PM support (Disable)
  --> Processor family
      --> Core 2/newer Xeon (if "cat /proc/cpuinfo | grep family" returns 6, set as Generic otherwise)
  // Xenomai will issue a warning about CONFIG_MIGRATION, disable those in this order
  --> Transparent Hugepage Support (Disable)
  --> Allow for memory compaction (Disable)
  --> Contiguous Memory Allocation (Disable)
  --> Allow for memory compaction
    --> Page Migration (Disable)
* Device Drivers
  --> Staging drivers
      --> Unisys SPAR driver support
         --> Unisys visorbus driver (Disable)
```


Now start the compilation
```sh
CONCURRENCY_LEVEL=$(nproc) make-kpkg --rootcmd fakeroot --initrd kernel_image kernel_headers
```

Install kernel
```sh
cd ..
sudo dpkg -i *.deb 

```

Allow non-root users

```sh
sudo addgroup xenomai --gid 1234
sudo addgroup root xenomai
sudo usermod -a -G xenomai $USER

```

## Grub configuration

To configure grub it is suggested to use grub-customizer

```sh
sudo nano /etc/default/grub
```
__Recommended Grub:__

```
  GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.4.133-xenomai-3.1"
  #GRUB_DEFAULT=saved
  #GRUB_SAVEDEFAULT=true
  # Comment the following lines
  #GRUB_HIDDEN_TIMEOUT=0
  #GRUB_HIDDEN_TIMEOUT_QUIET=true
  GRUB_TIMEOUT=5
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash xenomai.allowed_group=1234"
  GRUB_CMDLINE_LINUX=""
  after customization run
```
__If you have Intel HD Graphics____
```
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_rc6=0 i915.enable_dc=0 noapic xenomai.allowed_group=1234"
  # This removes powersavings from the graphics, that creates disturbing interruptions.
```
__If you have Intel Skylake Processor____
```
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_rc6=0 i915.enable_dc=0 xeno_nucleus.xenomai_gid=1234 nosmap"

```
```sh
sudo update-grub
sudo reboot
```

__Test if installed__:
```sh
uname -a
# Returns: Linux 5.4.133-xenomai-3.1 #2 SMP Sun Sept 5 12:25:57 PST 2021 x86_64 x86_64 x86_64 GNU/Linux
dmesg | grep Xenomai
# Returns:
#[    1.454252] [Xenomai] scheduling class idle registered.
#[    1.454253] [Xenomai] scheduling class rt registered.
#[    1.454400] I-pipe: head domain Xenomai registered.
#[    1.456208] [Xenomai] Cobalt v3.1 (xxx)

```

## Installing Xenomai User space libraries

```sh
cd xenomai-3
./configure --with-pic --with-core=cobalt --enable-smp --disable-tls #--disable-clock-monotonic-raw
make -j`nproc`
sudo make install
```

```sh
echo '
### Xenomai
export XENOMAI_ROOT_DIR=/usr/xenomai
export XENOMAI_PATH=/usr/xenomai
export PATH=$PATH:$XENOMAI_PATH/bin:$XENOMAI_PATH/sbin
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$XENOMAI_PATH/lib/pkgconfig
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$XENOMAI_PATH/lib
export OROCOS_TARGET=xenomai
' >> ~/.xenomai_rc

echo 'source ~/.xenomai_rc' >> ~/.bashrc
source ~/.bashrc
```
