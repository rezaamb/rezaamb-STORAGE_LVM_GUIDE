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

در دیسک‌های GPT، هدر پشتیبان در آخرین سکتورهای دیسک نگهداری می‌شود. وقتی دیسک مجازی بزرگ می‌شود، این هدر در وسط دیسک می‌افتد. اجرای دستور parted ... print متوجه این جابجایی شده و به صورت تعاملی درخواست Fix را صادر می‌کند.
```

