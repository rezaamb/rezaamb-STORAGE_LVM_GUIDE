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
