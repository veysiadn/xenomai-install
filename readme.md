# Xenomai real-time patch installation

## Guides

- [Xenomai docs](http://xenomai.org/installing-xenomai-3-x/#Installation_steps)
- [Readthedocs.io](http://rtt-lwr.readthedocs.io/en/latest/rtpc/xenomai.html)
- [Xenomai 3.0.5](http://rtt-lwr.readthedocs.io/en/latest/rtpc/xenomai3.html)

## Download links

- [Linux kernel](https://www.kernel.org/pub/linux/kernel/v4.x/)
- [Xenomai](http://xenomai.org/downloads/xenomai/stable/)
- [I-pipe](http://xenomai.org/downloads/ipipe/v4.x/x86/)


## Commands

### Prerequisites

```sh
sudo apt install -y kernel-package qt5-default libssl-dev git autoconf libtool automake curl
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
wget https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.9.51.tar.gz
tar xfv linux-4.9.51.tar.gz
wget http://xenomai.org/downloads/ipipe/v4.x/x86/ipipe-core-4.9.51-x86-4.patch

echo
echo "Patching Linux Kernel"
cd linux-4.9.51
./../xenomai-3/scripts/prepare-kernel.sh --arch=x86_64 --ipipe=../ipipe-core-4.9.51-x86-4.patch
cd ..


```

Now configure the kernel
```sh
yes "" | make oldconfig
make xconfig
```

__Recommended options:__

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
sudo dpkg -i linux-headers-4.9.51-xenomai-3.0.6_4.9.51-xenomai-3.0.6-10.00.Custom_amd64.deb linux-image-4.9.51-xenomai-3.0.6_4.9.51-xenomai-3.0.6-10.00.Custom_amd64.deb 

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
sudo add-apt-repository ppa:danielrichter2007/grub-customizer
sudo apt-get update
sudo apt-get install -y grub-customizer

```

after customization run

```sh
sudo update-grub
sudo reboot
```

__Test if installed__:
```sh
uname -a
# Returns: Linux polena 4.9.51-xenomai-3.0.6 #2 SMP Sun Mar 4 12:25:57 PST 2018 x86_64 x86_64 x86_64 GNU/Linux
dmesg | grep Xenomai
# Returns:
#[    1.454252] [Xenomai] scheduling class idle registered.
#[    1.454253] [Xenomai] scheduling class rt registered.
#[    1.454400] I-pipe: head domain Xenomai registered.
#[    1.456208] [Xenomai] Cobalt v3.0.6 (Stellar Parallax) 

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

## Installing Balena

```sh
sh balena-install.sh
```

---

There are two options to do RT scheduling within a container:

Add the SYS_NICE capability

docker run --cap-add SYS_NICE ...

Use privileged mode with --privileged flag

docker run --privileged ...

Privileged mode is said to be insecure, so option 1 would be best to only add the capability you need.

You may also have to enable realtime scheduling in your sysctl if you are running as the root user (default for Docker container):

sysctl -w kernel.sched_rt_runtime_us=-1
To make that permanent (update your image):

echo 'kernel.sched_rt_runtime_us=-1' > /etc/sysctl.conf
https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities
