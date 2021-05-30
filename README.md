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
```bash
Linux ip-10-0-3-34.ec2.internal 5.4.91-rt50 #1 SMP PREEMPT_RT Tue May 18 03:20:59 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux
```

This document assumes the Amazon Linux 2 with the RT-50 patch. You can find how to apply the real-time patch to Amazon Linux 2 [here](https://github.com/krsnucc21/AL2-RT-Patch)

Next, check the boot command line from the 'cmdline' file. Please carefully see the cmdline configured in a proper way.
```bash
cat /proc/cmdline 
```

The cmdline used through this DPDK install procedure looks like the following:
```bash
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
```bash
/boot/vmlinuz-5.4.91-rt50
```

After rebooting the machine, you can check the cmdline is correct.
```bash
cat /proc/cmdline
```

The result would be:
```bash
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
