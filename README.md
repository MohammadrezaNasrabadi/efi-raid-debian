# efi-raid-debian
A simple procedure for making the efi partition of debian operating system highly available.

## Overview
The EFI System Partition (ESP) is a partition on the storage that stores boot files for the Unified Extensible Firmware Interface (UEFI). The ESP is essential for an operating system to boot successfully. If the disk containing the ESP is corrupted, then We won't be able to boot the system easily; possibly We have to attach another rescue ISO file and recover the system or another solutions.

Here, We are going to provide a procedure to redundant the ESP across another existing disks; So, if the main disk containing the ESP was corrupted, the OS will be booted on another reboots without any problems.

**Note:**
This procedure is tested on `Debian` Operating System; but this solution may be also helpful for another distributions.

## Assumptions
Before We go ahead, let's describe the test environment so the procedure could be more clarify.

- Consider We have 2 disk, in type of NVMe.

- Consider the root filesystem is mounted and placed on a RAID array. The reason is because We should have redundancy for OS files as well.
---

The below is the details regarding how the disks are configured.

```bash
lsblk
NAME        MAJ:MIN RM    SIZE RO TYPE   MOUNTPOINTS
nvme0n1     259:0    0    3.5T  0 disk   
├─nvme0n1p1 259:2    0      1G  0 part   /boot/efi
├─nvme0n1p2 259:4    0     40G  0 part   
  └─md1       9:1    0     40G  0 raid1  /

nvme1n1     259:1    0    3.5T  0 disk   
├─nvme1n1p1 259:5    0      1G  0 part   
├─nvme1n1p2 259:9    0     40G  0 part   
│ └─md1       9:1    0     40G  0 raid1  /
```

As You can see, each NVMe disk is divided into 2 partitions and the first partition of the first disk is assigned as ESP. the second partition of disks is assigned as a RAID array that is used for root filesystem.

Now, We want to somehow make the `nvmen1p1` partition to be as same as `nvme0n1p1`; So We would have a copy version of the EFI on another disk and can ensure that We won't loose the efi data.

We choose RAID 1 solution to keep the efi data redundant and be up to date when updating the packages or boot files if is necessary.

At first, for making the ESP be raided, We should unmount the /boot/efi mountpoint.

```bash
umount /boot/efi
```

Now, We are ready to make a RAID 1 array.

```bash
yes | mdadm --create /dev/md2 --level=1 --metadata=0.90 --raid-devices=2 /dev/nvme0n1p1 /dev/nvme1n1p1
```

Please note that the metadata version for RAID array must be set to `0.9` to be compatible for ESP.

Now, We have a raid array that is going to be used as ESP. It is time to mount back the new RAID device on the `efi` mountpoint.

```bash
mount /dev/md2 /boot/efi
```

But if we check the list of files within the `/boot/efi` directory, We will see that there is not any file regarding for EFI. By issuing the below command, the EFI related files will be overriden on the newly created RAID device.

```bash
updateinitramfs -u
```

After issuing the above command, the `/boot/efi` mountpoint will be fulfilled.

By issuing the below command, We can check that the boot entries are updated so that the second disk is added as new boot entry.

```bash
efibootmgr -v
Boot0000* debian        HD(1,GPT,73548df2-c59d-4ec4-b02e-3e70c5ec2a49,0x800,0x200000)/File(\EFI\DEBIAN\SHIMX64.EFI)
Boot0003* UEFI: Built-in EFI Shell      VenMedia(5023b95c-db26-429b-a648-bd47664c8012)..BO
Boot0005* debian        HD(1,GPT,73548df2-c59d-4ec4-b02e-3e70c5ec2a49,0x800,0x200000)/File(\EFI\DEBIAN\GRUBX64.EFI)..BO
```

```bash
lsblk
NAME        MAJ:MIN RM    SIZE RO TYPE   MOUNTPOINTS
nvme0n1     259:0    0    3.5T  0 disk   
├─nvme0n1p1 259:2    0      1G  0 part
│ └─md2       9:4    0 1023.9M  0 raid1  /boot/efi
├─nvme0n1p2 259:4    0     40G  0 part   
  └─md1       9:1    0     40G  0 raid1  /

nvme1n1     259:1    0    3.5T  0 disk   
├─nvme1n1p1 259:5    0      1G  0 part
│ └─md2       9:4    0 1023.9M  0 raid1  /boot/efi
├─nvme1n1p2 259:9    0     40G  0 part   
│ └─md1       9:1    0     40G  0 raid1  /
```

Now, because the block device of ESP is changed from a single partition to a RAID array, some service files such as `fsck` service must be changed. `fsck` service will check all of the entries within the `/etc/fstab` file are presented during the boot process.

By reloading the systemctl, all of the necessary changes will be applied automatically.

```bash
systemctl daemon-reload
```

## How to test?
By removing all of the partitions within the first disk, We simulate the situation of disk corruption. We can clear all of the first disk partition using the `parted` command, or another tools. After clearing partitions and rebooting the system, The OS will be booted on the second disk successfully.