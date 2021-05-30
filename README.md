# DPDK-AL2-RT

## Overview

This document provides a step-by-step procedure to build DPDK on Amazon Linux 2 for Graviton2 processor. Basically, the general process for Amazon Linux 2 can be found at [this link](https://github.com/amzn/amzn-drivers/tree/master/userspace/dpdk) and the additional steps for VFIO on AL2 are described [here](https://github.com/amzn/amzn-drivers/tree/master/userspace/dpdk/enav2-vfio-patch). However, for the linux kernel with the real-time patch, it needs more special steps. Focusing on DPDK on the top of a real-time patched AL2 on Graviton2, This document provides build, install, and test commands to run DPDK.

This document assumes huge memory pages have been properly configured before start. The information how to set up huge pages in Linux can be found at [this link](https://aws.amazon.com/premiumsupport/knowledge-center/configure-hugepages-ec2-linux-instance/) You can also refer to the following example commands:

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
``` bash
cat /proc/cmdline 
```

The cmdline used through this DPDK install procedure looks like the following:
```bash
BOOT_IMAGE=/boot/vmlinuz-5.4.91-rt50 root=UUID=53a36bec-2f52-4183-8f7f-3acfb060d4b3 ro console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0 no_timer_check rcu_nocbs=0-7 rcu_nocb_poll=1 nohz=on nohz_full=0-7 isolcpus=0-7 irqaffinity=8-15 selinux=0 enforcing=0 noswap default_hugepagesz=1G hugepagesz=1G hugepages=30 mce=off audit=0 crashkernel=auto nmi_watchdog=0 fsck.mode=force fsck.repair=yes skew_tick=1 softlockup_panic=0 idle=poll nosoftlockup pcie_aspm.policy=performance
```

## Step 2: Config IOMMU passthrough mode
