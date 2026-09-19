🌳 Architectural Tree & Decision Workflow
```bash
Storage Management Workflow
│
├── [1] Initial Setup (Raw Provisioning)
│   └── Raw Disk (/dev/sdb) ──► pvcreate ──► vgcreate ──► lvcreate ──► mkfs.ext4 / xfs
│
├── [2] Scale-Up (Expand Existing Attached Disk via Hypervisor)
│   │
│   ├── Step 0: Discovery (lsblk, pvs, vgs, lvs, df -Th)
│   ├── Step 1: Rescan SCSI Bus (echo 1 > rescan)
│   │
│   ├── Decision: Does the disk have a partition table?
│   │   ├── YES (Partitioned: e.g., sda3, sdb1, sdc1)
│   │   │   ├── GPT Header Check & Fix (parted print)
│   │   │   ├── Expand Partition (growpart /dev/sdX <part_num>)
│   │   │   └── Update PV (pvresize /dev/sdX<part_num>)
│   │   │
│   │   └── NO (Raw Disk: e.g., sda, sdb, sdc)
│   │       └── Update PV directly (pvresize /dev/sdX)
│   │
│   └── Step 3: Expand Logical Volume & Filesystem (lvextend -r)
│
└── [3] Scale-Out (Add New Disk Device)
    ├── Step 1: Rescan SCSI Bus & Identify New Device (/dev/sdX)
    ├── Step 2: Initialize PV (pvcreate /dev/sdX)
    ├── Step 3: Extend Volume Group (vgextend <vg_name> /dev/sdX)
    └── Step 4: Expand Logical Volume & Filesystem (lvextend -r)
```
📋 Quick Reference Command Matrix
```bash
Scenario	Layer	Primary Command	Alternative / Notes
Rescan Bus	Kernel / SCSI	echo 1 > /sys/class/block/sdX/device/rescan	for host in /sys/class/scsi_host/...
Partition Fix	Disk Table (GPT)	parted /dev/sdX print	Fixes backup header at end of disk
Partition Resize	Partition Table	growpart /dev/sdX <part_num>	parted /dev/sdX resizepart <num> 100%
PV Resize	LVM Physical	pvresize /dev/sdX<part_num> (or /dev/sdX for raw)	Syncs PV size with underlying block size
LV + FS Resize	LVM Logical + FS	lvextend -r -l +100%FREE /dev/<vg>/<lv>	-r handles resize2fs / xfs_growfs
```


1. Initial Storage Bootstrap (From Scratch - Raw Disk)

این سناریو برای راه‌اندازی دیسک جدید به صورت Raw بدون افزودن لایه پیچیدگی پارتیشن‌بندی استفاده می‌شود.
```bash
# 1. Initialize disk as an LVM Physical Volume
sudo pvcreate /dev/sdb

# 2. Create a Volume Group containing the PV
sudo vgcreate vg_data /dev/sdb

# 3. Create a Logical Volume allocating 100% of the VG free space
sudo lvcreate -n lv_data -l 100%FREE vg_data

# 4. Format the Logical Volume with ext4 filesystem
sudo mkfs.ext4 /dev/vg_data/lv_data

# 5. Persist mount point in /etc/fstab (Recommended: use UUID via blkid)
# Example: UUID=xxxx-xxxx /data ext4 defaults 0 0
```

2. Scenario A: Expanding an Existing Disk (Scale-Up)

Step 0: Discovery & Layer Auditing

همیشه قبل از تغییر ساختار، وضعیت تمام لایه‌ها را مستند و بررسی کنید:

```bash
# Block devices, types, filesystems and mountpoints
lsblk -f
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS

# Partition geometry and sector layout
sudo fdisk -l /dev/sda
sudo fdisk -l /dev/sdb
sudo fdisk -l /dev/sdc

# LVM layers status
sudo pvs -o pv_name,vg_name,pv_size,pv_free
sudo vgs -o vg_name,pv_count,lv_count,vg_size,vg_free
sudo lvs -o lv_name,vg_name,lv_size,lv_attr

# Filesystem usage
df -Th
```

Step 1: SCSI Bus Rescan (Kernel Detection)

پس از افزایش سایز در مجازی‌ساز (vSphere / Proxmox / KVM / Cloud)، کرنل باید بلاک‌دیوایس را بازخوانی کند:

```bash
# Per-device rescan (Fast & targeted)
echo 1 | sudo tee /sys/class/block/sda/device/rescan
echo 1 | sudo tee /sys/class/block/sdb/device/rescan
echo 1 | sudo tee /sys/class/block/sdc/device/rescan

# Full SCSI bus rescan (Fallback for all controllers)
for host in /sys/class/scsi_host/host*/scan; do echo "- - -" | sudo tee "$host" > /dev/null; done
```
Branch 1: Partitioned Disk (e.g., sda3, sdb1, sdc1)

```
⚠️ GPT Backup Header Notice:
```

در دیسک‌های GPT، هدر پشتیبان در آخرین سکتورهای دیسک نگهداری می‌شود. وقتی دیسک مجازی بزرگ می‌شود، این هدر در وسط دیسک می‌افتد. اجرای دستور parted ... print متوجه این جابجایی شده و به صورت تعاملی درخواست Fix را صادر می‌کند.



1. Check & Fix GPT Header
```bash
sudo parted /dev/sda print
sudo parted /dev/sdb print
sudo parted /dev/sdc print
```

2. Expand Partition Table

```bash
# Primary & Recommended Method (cloud-guest-utils)
sudo growpart /dev/sda 1
sudo growpart /dev/sda 3
sudo growpart /dev/sdb 1
sudo growpart /dev/sdc 1

# Alternative Method using parted directly
sudo parted /dev/sda resizepart 1 100%
sudo parted /dev/sda resizepart 3 100%
sudo parted /dev/sdb resizepart 1 100%
sudo parted /dev/sdc resizepart 1 100%
```


3. Update Physical Volume (PV)

```bash
sudo pvresize /dev/sda1
sudo pvresize /dev/sda3
sudo pvresize /dev/sdb1
sudo pvresize /dev/sdc1
```


Branch 2: Raw Disk (No Partition Table)
```💡 Raw Disk Benefits:```
هیچ نیازی به growpart، parted، دستکاری سکتورها یا رفع خطای GPT Header نیست. این فرایند ظرف ۲ ثانیه و بدون کوچک‌ترین ریسک به پایان می‌رسد.

```bash
# 1. Rescan device
echo 1 | sudo tee /sys/class/block/sda/device/rescan
echo 1 | sudo tee /sys/class/block/sdb/device/rescan
echo 1 | sudo tee /sys/class/block/sdc/device/rescan

# 2. Resize PV directly pointing to the whole block device
sudo pvresize /dev/sda
sudo pvresize /dev/sdb
sudo pvresize /dev/sdc
```

3. Scenario B: Adding a Brand New Physical Disk (Scale-Out)

هنگامی که یک هارد دیسک مجازی یا فیزیکی جدید به سیستم متصل شده و هدف، تزریق آن به استخر ذخیره‌سازی فعلی است:


```bash
# 1. Scan for the new physical disk
echo 1 | sudo tee /sys/class/block/sdb/device/rescan
echo 1 | sudo tee /sys/class/block/sdc/device/rescan
for host in /sys/class/scsi_host/host*/scan; do echo "- - -" | sudo tee "$host" > /dev/null; done

# 2. Initialize new physical disks as PVs
sudo pvcreate /dev/sdb
sudo pvcreate /dev/sdc

# 3. Extend existing Volume Group with newly added PVs
sudo vgextend vgdocker /dev/sdb
sudo vgextend vgdocker /dev/sdc
```

4. Expanding Logical Volume & Filesystem (Common Final Step)

سوییچ -r (--resizefs) در دستور lvextend فرآیند گسترش حجم منطقی و فایل‌سیستم زیربنایی را به صورت کاملاً آنلاین و اتمیک ترکیب می‌کند.

```bash
# Option 1: Allocate 100% of all available free extents in the VG
sudo lvextend -r -l +100%FREE /dev/vgdocker/lvdocker

# Option 2: Add a fixed amount (e.g., +50GB)
sudo lvextend -r -L +50G /dev/vgdocker/lvdocker

# Option 3: Allocate a percentage of the total Volume Group capacity
sudo lvextend -r -l +50%VG /dev/vgdocker/lvdocker
```


5. Manual Filesystem Expansion (Fallback Mode)

اگر به هر دلیلی گسترش LV را بدون سوییچ -r انجام دادید، سیستم‌فایل را به صورت دستی و با توجه به نوع فرمت آن افزایش دهید:

ext4 / ext3 / ext2

با آدرس بلاک‌دیوایس LVM اجرا شود:

```bash
sudo resize2fs /dev/mapper/vgdocker-lvdocker
```

XFS

باید مستقیماً به مسیر Mount Point فعال داده شود، نه آدرس Device:
```bash
sudo xfs_growfs /var/lib/docker
```


