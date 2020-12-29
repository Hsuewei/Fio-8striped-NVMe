Fio-8striped-NVMe
===
Fio test on eight striped NVMe at raid-0 with mdadm

Table of Contents
===
* [Introduction](#introduction)
* [Build Fio](#build-fio)  
* [Environment](#environment)
* [Learning](#learning)
## Introduction
This repos contains multiple rounds of `fio` test result and configs of four Intel P4510 ssds's striped volumn.
## Build Fio
Can refer to [here](https://hackmd.io/vYiT2eZoRDac0K8NU7SA-A#build) for build fio on RHEL7.5
## Environment
1. Use 8 Intel P4510 [4TB](https://ark.intel.com/content/www/us/en/ark/products/122579/intel-ssd-dc-p4510-series-4-0tb-2-5in-pcie-3-1-x4-3d2-tlc.html), and divide them to two groups(4 drives each). Each group create a storage pool with `mdadm`:
  ``` bash
  # create partition
  for i in {0..7}; do parted -s -a optimal /dev/nvme${i}n1 mklabel gpt mkpart primary xfs 1MiB 100%
  # /dev/md0
  mdadm --create --verbose /dev/md0 --level=raid0 --raid-devices=4 /dev/nvme0n1p1 /dev/nvme2n1p1 /dev/nvme4n1p1 /dev/nvme6n1p1
  # /dev/md1
  mdadm --create --verbose /dev/md1 --level=raid0 --raid-devices=4 /dev/nvme1n1p1 /dev/nvme3n1p1 /dev/nvme5n1p1 /dev/nvme7n1p1
  mdadm --detail --scan >> /etc/mdadm.conf
  ```
2. Format and mount
  ``` bash
  mkdir -p /opt/fio/{t0,t1}
  mkfs.xfs -f /dev/md0
  mkfs.xfs -f /dev/md1
  mount -o noatime,nodiratime,discard /dev/md0 /opt/fio/t0
  mount -o noatime,nodiratime,discard /dev/md0 /opt/fio/t1
  ```
## Example
``` bash
./bin/fio /opt/fio/configs/MS-28k-r-r --output /opt/fio/results/MS-28k-r-r.log
```
## Appendix: Clean the floor
1. stop `mdadm`
   ``` bash
   mdadm --stop /dev/md0
   mdadm --stop /dev/md1
   for i in {0..7}; do mdadm --zero-superblock /dev/nvme${i}n1p1; done
   ```
2. clean partition with `dd`
   ``` bash
   for i in {0..7}; do dd if=/dev/zero of=/dev/nvme${i}n1 bs=512 count=10000; done
   ```
3. remove `mdadm.conf`
   ``` bash
   sed -i.bak "\@^ARRAY@d" /etc/mdadm.conf
   ```
## Learning:
### Use `filename=` or `directory=` ?
Typically, most fio benchmarkers prefer to use `filename=/dev/certaindevice`, because directly performing test on raw device without partition or filesystem will give greater result. [MS-28k-r-r-11], [MS-28k-r-r-15] and [MS-28k-r-r-16] give some insight:
1. [MS-28k-r-r-11] use `directory=/opt/fio/t0` and, according to config, this will create 8 100g-file under */opt/fio/t0* 
2. [MS-28r-r-r-15] use `filename=/dev/md0` directly on raw device. However, with the fio files created by round 11, we get close number in IOPS as round 11
3. [MS-28k-r-r-16], I removed all fio files under the mount point */opt/fio/t0*, and use `filename=/dev/md0` directly on raw device.

round | IOPS
---|----
11 | 137K
15 | 137K
16 | 238K

### Peak performance for single intel P4510
1. [MS-28-r-r-20] use 8 NVMe 8 threads(numjobs=8) 
2. [MS-28k-r-r-11] use 4 NVMe 8 threads(numjobs=8)
