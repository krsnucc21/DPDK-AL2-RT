# DPDK-AL2-RT

## Overview

This document provides a step-by-step procedure to build DPDK on Amazon Linux 2 for Graviton2 processor. Basically, the general process for Amazon Linux 2 can be found at [this link](https://github.com/amzn/amzn-drivers/tree/master/userspace/dpdk) and the additional steps for VFIO on AL2 are described [here](https://github.com/amzn/amzn-drivers/tree/master/userspace/dpdk/enav2-vfio-patch). However, for the linux kernel with the real-time patch, it needs more special steps. Focusing on DPDK on the top of a real-time patched AL2 on Graviton2, This document provides build, install, and test commands to run DPDK.

This document assumes huge memory pages have been properly configured before start. The information how to set up huge pages in Linux can be found at [this](https://aws.amazon.com/premiumsupport/knowledge-center/configure-hugepages-ec2-linux-instance/) You can also refer to the following example commands:

```bash
# example commands to set up huge memory pages
echo 'vm.nr_hugepages=20480' > /etc/sysctl.d/hugepages.conf
sysctl -w vm.nr_hugepages=20480
mkdir /mnt/huge
chmod 777 /mnt/huge
vi /etc/fstab
# Add the following line to the end of the file:
# 'huge /mnt/huge hugetlbfs defaults 0 0'
mount -a
```

## Step 1: Check the kernel version and cmdline

First check the version of the RT-patched kernel by using 'uname -a'.

```bash
uname -a
```

The version should be:
```
Linux ip-10-0-3-34.ec2.internal 5.4.91-rt50 #1 SMP PREEMPT_RT Tue May 18 03:20:59 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux
```

This document assumes the Amazon Linux 2 with the RT-50 patch. You can find how to apply the real-time patch to Amazon Linux 2 [here](https://github.com/krsnucc21/AL2-RT-Patch)

Next, check the boot command line from the 'cmdline' file. Please carefully see the cmdline configured in a proper way.
```bash
cat /proc/cmdline 
```

The cmdline used through this DPDK install procedure looks like the following:
```
BOOT_IMAGE=/boot/vmlinuz-5.4.91-rt50 root=UUID=53a36bec-2f52-4183-8f7f-3acfb060d4b3 ro console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0 no_timer_check rcu_nocbs=0-7 rcu_nocb_poll=1 nohz=on nohz_full=0-7 isolcpus=0-7 irqaffinity=8-15 selinux=0 enforcing=0 noswap default_hugepagesz=1G hugepagesz=1G hugepages=30 mce=off audit=0 crashkernel=auto nmi_watchdog=0 fsck.mode=force fsck.repair=yes skew_tick=1 softlockup_panic=0 idle=poll nosoftlockup pcie_aspm.policy=performance
```

## Step 2: Config IOMMU passthrough mode

On AWS, the metal instances are supporting IOMMU for Graviton2. IOMMU should be enabled by default. Unfortunately, igb_uio and vfio-pci doesn't supporting either IOMMU or SMMU, which is implementation of IOMMU for arm64 architecture, so to use DPDK with ENA on those hosts, one must disable IOMMU. This can be done by updating GRUB_CMDLINE_LINUX in file /etc/default/grub with the extra boot argument.

```bash
sudo vi /etc/default/grub
# To 'GRUB_CMDLINE_LINUX', add a line of 'iommu.passthrough=1'
```

To apply this grub change, use grub2 and grubby commands as follows:
```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo grubby --set-default /boot/vmlinuz-5.4.91-rt50
sudo grubby --default-kernel
```

The last command will show the default kernel version to boot this machine.
```
/boot/vmlinuz-5.4.91-rt50
```

After rebooting the machine, you can check the cmdline is correct.
```bash
cat /proc/cmdline
```

The result would be:
```
BOOT_IMAGE=/boot/vmlinuz-5.4.91-rt50 root=UUID=53a36bec-2f52-4183-8f7f-3acfb060d4b3 ro console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0 no_timer_check rcu_nocbs=0-7 rcu_nocb_poll=1 nohz=on nohz_full=0-7 isolcpus=0-7 irqaffinity=8-15 selinux=0 enforcing=0 noswap default_hugepagesz=1G hugepagesz=1G hugepages=30 mce=off audit=0 crashkernel=auto nmi_watchdog=0 fsck.mode=force fsck.repair=yes skew_tick=1 softlockup_panic=0 idle=poll nosoftlockup pcie_aspm.policy=performance iommu.passthrough=1
```

## Step 2: Apply AWS patch to VFIO

VFIO-PCI driver does not support write combine. To activate this feature, the patch that checks if PCI BAR is prefetchable must be added. The basic steps are the same as in [this page](https://github.com/amzn/amzn-drivers/tree/master/userspace/dpdk/enav2-vfio-patch), but with the RT-patched kernel, we should use our own patched kernel source code to build the VFIO drivers.

First, download the patch source code from the GitHub.
```bash
git clone https://github.com/amzn/amzn-drivers.git
cd amzn-drivers/userspace/dpdk/enav2-vfio-patch/
```

And, copy the patched kernel source code under the 'tmp/linux-5.4.91' directory.
```bash
mkdir tmp
cd tmp
cp -R ~/rpmbuild/BUILD/kernel-5.4.91.amzn2/linux-5.4.91-41.139.amzn2.aarch64 linux-5.4.91
cd ..
```

Then, change the shell script to apply the patches to our kernel code.
```bash
vi get-vfio-with-wc.sh
```

The following code changes should be applied:
```
"get-vfio-with-wc.sh" line 86 of 200
function download_kernel_src {
        bold "[1] Downloading prerequisites..."
        #rm -rf "${TMP_DIR}"
        mkdir -p "${TMP_DIR}"
        cd "${TMP_DIR}"

        #if apt-get -v >/dev/null 2>/dev/null; then
        #       download_kernel_src_apt
        #else
        #       download_kernel_src_yum
        #fi
        cd linux-*
}
```

Run the patch command to build and install VFIO drivers
```bash
sudo ./get-vfio-with-wc.sh
```

You may need to run the following commands to set up the VFIO drivers.
```bash
sudo su
echo vfio > /etc/modules-load.d/vfio.conf
echo vfio_pci > /etc/modules-load.d/vfio_pci.conf
echo "options vfio enable_unsafe_noiommu_mode=1" > /etc/modprobe.d/vfio-noiommu.conf
```

Reboot your machine and see the changes have been applied correctly by the following commands:
```bash
lsmod
cat /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
```

## Step 3: Build and install DPDK

Now, download the code of DPDK. v20.02 is recommended; as of today, that version is the latest one with the patches for Amazon Linux 2.
```bash
% git clone git://dpdk.org/dpdk
% cd dpdk
% git checkout v20.02
```

Since we have downloaded the Amazon Linux 2 patch for DPDK in the step 2, simply apply the patches to the DPDK code.
```bash
% git am ~/amzn-drivers/userspace/dpdk/20.02/*.patch
```

Then, the patches will be applied as follows:
```
Applying: net/ena: ensure Rx buffer size is at least 1400B
Applying: net/ena/base: make allocation macros thread-safe
Applying: net/ena/base: prevent allocation of zero sized memory
Applying: net/ena: set IO ring size to valid value
Applying: net/ena: remove memory barriers before doorbells
Applying: net/ena: limit refill threshold by fixed value
Applying: net/ena: fix build for O1 optimization
```

It is ready to build and install. Please run the following commands to install DPDK:
```bash
make config T=arm64-armv8a-linuxapp-gcc
make
sudo make install
```

The installed DPDK commands are found in the '/usr/local/bin' directory.
```base
ls /usr/local/bin/ -l
```

You can find new commands like 'testpmd'.
```
-rwxr-xr-x 1 root root 14766032 May 27 15:44 testpmd
-rwxr-xr-x 1 root root 13864664 May 27 15:44 dpdk-procinfo
-rwxr-xr-x 1 root root 13864480 May 27 15:44 dpdk-pdump
-rwxr-xr-x 1 root root 13862064 May 27 15:44 testsad
-rwxr-xr-x 1 root root 13934672 May 27 15:44 testbbdev
-rwxr-xr-x 1 root root 13864056 May 27 15:44 dpdk-test-compress-perf
-rwxr-xr-x 1 root root 13871080 May 27 15:44 dpdk-test-crypto-perf
-rwxr-xr-x 1 root root 13938800 May 27 15:44 dpdk-test-eventdev
lrwxrwxrwx 1 root root       39 May 27 15:45 dpdk-pmdinfo -> ../share/dpdk/usertools/dpdk-pmdinfo.py
```

## Step 4: Configure a DPDK interface with the VFIO driver

With the VFIO driver we've patched, a DPDK network interface now can be configured. First, check if the driver and interfaces are available by the following command:
```bash
usertools/dpdk-devbind.py --status
```

If there are network interfaces available, the result may look like the following:
```
Network devices using kernel driver
===================================
0000:00:05.0 'Elastic Network Adapter (ENA) ec20' if=eth0 drv=ena unused=vfio-pci *Active*
0000:00:06.0 'Elastic Network Adapter (ENA) ec20' if=eth1 drv=ena unused=vfio-pci 
```

Then, we set up the interface 'eth1' as a DPDK interface by the following command:
```bash
sudo usertools/dpdk-devbind.py --bind=vfio-pci eth1
```

You can check if that is set up correctly by using the previous 'dpdk-devbind.py' command. The result would look like the following:
```
Network devices using DPDK-compatible driver
============================================
0000:00:06.0 'Elastic Network Adapter (ENA) ec20' drv=vfio-pci unused=ena

Network devices using kernel driver
===================================
0000:00:05.0 'Elastic Network Adapter (ENA) ec20' if=eth0 drv=ena unused=vfio-pci *Active*
```

You also can do a test with the new interface. Please run the following test command:
```bash
sudo /usr/local/bin/testpmd -- -i
```

The 'testpmd' runs in the interactive mode and the following command shows the DPDK interface information.
```bash
show port info 0
```

The result may look like this:
```
********************* Infos for port 0  *********************
MAC address: 0A:CB:C7:53:15:ED
Device name: 0000:00:06.0
Driver name: net_ena
Connect to socket: 0
memory allocation on the socket: 0
Link status: up
Link speed: 0 Mbps
Link duplex: full-duplex
MTU: 1500
Promiscuous mode: disabled
Allmulticast mode: disabled
```

## (Optional) Step 5: Build Pktgen
