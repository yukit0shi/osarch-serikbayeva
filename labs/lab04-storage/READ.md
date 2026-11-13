# Lab 4 — Partition, mirror, fail, recover

Aisulu · Date: 13 November 2026 · Hypervisor: Oracle VirtualBox

## Windows

Disk 1 (1 GB, added as `lab04-win`) was RAW. I initialised it as GPT, created one partition on the whole disk, formatted it NTFS with the label LAB04 and the system assigned letter E.

```powershell
Initialize-Disk -Number 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel LAB04
```

```
PS C:\WINDOWS\system32> Get-Disk | Select Number, FriendlyName, PartitionStyle, Size

Number FriendlyName    PartitionStyle        Size
------ ------------    --------------        ----
     0 VBOX HARDDISK   GPT            85899345920
     1 VBOX HARDDISK   GPT             1073741824

PS C:\WINDOWS\system32> Get-Volume | Where FileSystemLabel -eq 'LAB04'

DriveLetter FriendlyName FileSystemType DriveType HealthStatus OperationalStatus SizeRemaining      Size
----------- ------------ -------------- --------- ------------ ----------------- -------------      ----
E           LAB04        NTFS           Fixed     Healthy      OK                    989.98 MB 1007.93 MB
```

MBR or GPT: GPT. I chose it because Windows 11 starts through UEFI, which needs GPT, and GPT also stores a backup copy of the partition table and protects it with checksums, whereas MBR cannot handle disks above 2 TiB or more than four primary partitions.

## Linux — layout before and after

### Before

```
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
NAME                       SIZE TYPE FSTYPE      MOUNTPOINTS
loop0                     66.8M loop squashfs    /snap/core24/2124
loop1                     66.8M loop squashfs    /snap/core24/1643
loop2                      1.5M loop squashfs    /snap/hwctl/131
loop3                      1.5M loop squashfs    /snap/hwctl/123
loop4                     50.1M loop squashfs    /snap/snapd/27710
sda                         25G disk
├─sda1                       1M part
├─sda2                       2G part ext4        /boot
└─sda3                      23G part LVM2_member
  └─ubuntu--vg-ubuntu--lv 11.5G lvm  ext4        /
sdb                          1G disk
sdc                          1G disk
sdd                          1G disk
sr0                       1024M rom
```

### After

md0 is built on sdb1 and sdd1 and mounted on /mnt/mirror. sdc1 is the disk that was failed on purpose and replaced by sdd1; it still carries an old RAID superblock but belongs to no array.

```
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
NAME                       SIZE TYPE  FSTYPE            MOUNTPOINT
loop0                     66.8M loop  squashfs          /snap/core24/2124
loop1                     66.8M loop  squashfs          /snap/core24/1643
loop2                      1.5M loop  squashfs          /snap/hwctl/131
loop3                      1.5M loop  squashfs          /snap/hwctl/123
loop4                     50.1M loop  squashfs          /snap/snapd/27710
sda                         25G disk
├─sda1                       1M part
├─sda2                       2G part  ext4              /boot
└─sda3                      23G part  LVM2_member
  └─ubuntu--vg-ubuntu--lv 11.5G lvm   ext4              /
sdb                          1G disk
└─sdb1                    1022M part  linux_raid_member
  └─md0                   1021M raid1 ext4              /mnt/mirror
sdc                          1G disk
└─sdc1                    1022M part  linux_raid_member
sdd                          1G disk
└─sdd1                    1022M part  linux_raid_member
  └─md0                   1021M raid1 ext4              /mnt/mirror
sr0                       1024M rom
```

## What I did

Partition the three new disks (GPT, one partition each, marked for RAID):

```bash
for d in sdb sdc sdd; do
  sudo parted -s /dev/$d mklabel gpt mkpart primary 1MiB 100% set 1 raid on
done
```

Create the mirror from sdb1 and sdc1, then check it is healthy:

```bash
sudo apt install -y mdadm
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
cat /proc/mdstat
```

File system, mount, proof file and checksum:

```bash
sudo mkfs.ext4 -L mirror /dev/md0
sudo mkdir -p /mnt/mirror
sudo mount /dev/md0 /mnt/mirror
sudo chown $USER: /mnt/mirror
echo "Lab 4 proof file - ... - $(date)" > /mnt/mirror/proof.txt
head -c 20M /dev/urandom > /mnt/mirror/random.bin
cd /mnt/mirror && sha256sum proof.txt random.bin
```

Fail one disk with the software method (`--fail`, not by detaching the virtual disk), check the data is still readable and the checksums are the same:

```bash
sudo mdadm /dev/md0 --fail /dev/sdc1
sudo mdadm /dev/md0 --remove /dev/sdc1
cat /mnt/mirror/proof.txt
sha256sum proof.txt random.bin
```

Add the third disk and rebuild:

```bash
sudo mdadm /dev/md0 --add /dev/sdd1
cat /proc/mdstat
```

Make the array survive a reboot:

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

## Three states

| State | What /proc/mdstat showed | File |
| --- | --- | --- |
| Healthy | [2/2] [UU], sdb1 and sdc1 active sync, State: clean | evidence/array-healthy.txt |
| Degraded | [2/1] [U_], sdb1 only, State: clean, degraded | evidence/array-degraded.txt |
| Rebuilt | [2/2] [UU], sdd1 replaced sdc1, State: clean | evidence/array-rebuilt.txt |

Checksums before and after the failure: match (both files contain the same two SHA-256 sums, and `diff` printed "checksums match").

The rebuild of the 1 GB disk was already finished the first time I looked at /proc/mdstat after `--add`, so I did not catch a `recovery = ...%` line; the rebuilt state was saved as soon as it showed [2/2] [UU].

## RAID is not a backup

No, RAID 1 does not help here, because a deletion is written to both disks of the mirror at the same moment, so both copies lose the file. RAID only protects against a physical disk failure, not against human error, malware or corruption that gets mirrored as well. To get the file back we would need a separate backup or snapshot made before the deletion.

## What did not work the first time

1. In Ubuntu I pasted the partitioning loop together with the next command. `sudo` asked for the password in the middle of the paste and the following line `lsblk` was typed into the password prompt. I checked with `lsblk` and the partitions sdb1, sdc1 and sdd1 had been created correctly, so I did not repeat it. After that I ran `sudo -v` first before every block with sudo.
2. Two evidence files had a typo in the first line that shows the command: `$ cat/proc/mdstat` in array-healthy.txt and `random bin` in checksum-before.txt. The outputs under them were correct. I fixed only the command line (line 1) with `sed -i '1s|...|...|'`; the outputs and checksums were not edited. The first attempt at the second `sed` had a missing quote and the shell waited for more input until I pressed Ctrl+C.
