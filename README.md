# DPDK-AL2-RT

# Overview

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
