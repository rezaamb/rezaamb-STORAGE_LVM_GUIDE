# 🌳 Architectural Tree & Decision Workflow

```bash
Storage Management Workflow
│
├── [1] Initial Setup (Raw Provisioning)
│   └── Raw Disk (/dev/sdb)
│       └──► pvcreate
│           └──► vgcreate
│               └──► lvcreate
│                   └──► mkfs.ext4 / mkfs.xfs
│                       └──► Mount
│                           └──► /etc/fstab
│
├── [2] Scale-Up
│   │   └── Expand Existing Attached Disk via Hypervisor
│   │
│   ├── Step 0: Discovery
│   │   └── lsblk / pvs / vgs / lvs / df -Th
│   │
│   ├── Step 1: Rescan Storage
│   │   └── SCSI Bus Rescan
│   │
│   ├── Step 2: Determine Disk Layout
│   │   │
│   │   ├── [A] Partitioned Disk
│   │   │   └── Example: /dev/sda3 / /dev/sdb1 / /dev/sdc1
│   │   │       │
│   │   │       ├── Check GPT
│   │   │       │   └── parted /dev/sdX print
│   │   │       │
│   │   │       ├── Expand Partition
│   │   │       │   └── growpart /dev/sdX <part_num>
│   │   │       │
│   │   │       └── Resize PV
│   │   │           └── pvresize /dev/sdX<part_num>
│   │   │
│   │   └── [B] Raw Disk
│   │       └── Example: /dev/sdb /dev/sdc
│   │           │
│   │           └── Resize PV Directly
│   │               └── pvresize /dev/sdX
│   │
│   └── Step 3: Expand LV + Filesystem
│       └── lvextend -r
│
└── [3] Scale-Out
    │   └── Add Brand New Disk Device
    │
    ├── Step 1: Detect New Disk
    │   └── lsblk / dmesg / SCSI Rescan
    │
    ├── Step 2: Initialize PV
    │   └── pvcreate /dev/sdX
    │
    ├── Step 3: Extend VG
    │   └── vgextend <vg_name> /dev/sdX
    │
    └── Step 4: Expand LV + Filesystem
        └── lvextend -r
```

---

# 📋 Quick Reference Command Matrix

```text
Scenario              Layer                    Primary Command
──────────────────────────────────────────────────────────────────────────────────────────────
Rescan Device         Kernel / SCSI            echo 1 | tee /sys/class/block/sdX/device/rescan

Full SCSI Rescan      Kernel / SCSI            echo "- - -" > /sys/class/scsi_host/hostX/scan

Check Disk Layout     Block Device             lsblk -f
                                              fdisk -l /dev/sdX

Check GPT             Partition Table          parted /dev/sdX print

Resize Partition      Partition Table          growpart /dev/sdX <part_num>

Alternative           Partition Table          parted /dev/sdX resizepart <part_num> 100%

Resize PV              LVM Physical Volume     pvresize /dev/sdX<part_num>

Raw Disk PV Resize     LVM Physical Volume     pvresize /dev/sdX

Add New PV             LVM Physical Volume     pvcreate /dev/sdX

Extend VG              LVM Volume Group        vgextend <vg_name> /dev/sdX

Resize LV + FS         LVM + Filesystem        lvextend -r -l +100%FREE /dev/<vg>/<lv>

Resize LV Only         LVM Logical Volume      lvextend -l +100%FREE /dev/<vg>/<lv>

Resize ext4             Filesystem              resize2fs /dev/<vg>/<lv>

Resize XFS              Filesystem              xfs_growfs <mount_point>
```

---

# 1. Initial Storage Bootstrap

## From Scratch — Raw Disk

این سناریو زمانی استفاده می‌شود که یک دیسک جدید داریم و می‌خواهیم آن را مستقیماً به عنوان یک LVM Physical Volume استفاده کنیم.

در این مدل، دیسک بدون Partition Table مستقیماً به عنوان PV استفاده می‌شود.

```bash
Raw Disk
   │
   ▼
/dev/sdb
   │
   ▼
pvcreate
   │
   ▼
vgcreate
   │
   ▼
lvcreate
   │
   ▼
mkfs.ext4 / mkfs.xfs
   │
   ▼
Mount Point
```

### Step 0: Verify the Disk

⚠️ قبل از `pvcreate` حتماً مطمئن شوید که `/dev/sdb` دیسک صحیح است و اطلاعات مهمی روی آن وجود ندارد.

```bash
lsblk -f
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS

sudo fdisk -l /dev/sdb
```

### Step 1: Initialize Disk as LVM Physical Volume

```bash
sudo pvcreate /dev/sdb
```

### Step 2: Create Volume Group

```bash
sudo vgcreate vg_data /dev/sdb
```

### Step 3: Create Logical Volume

در این مثال کل فضای آزاد VG به LV اختصاص داده می‌شود:

```bash
sudo lvcreate -n lv_data -l 100%FREE vg_data
```

### Step 4: Create Filesystem

برای ext4:

```bash
sudo mkfs.ext4 /dev/vg_data/lv_data
```

یا برای XFS:

```bash
sudo mkfs.xfs /dev/vg_data/lv_data
```

### Step 5: Create Mount Point

```bash
sudo mkdir -p /data
```

### Step 6: Mount Filesystem

```bash
sudo mount /dev/vg_data/lv_data /data
```

### Step 7: Verify

```bash
lsblk -f
df -Th /data
```

### Step 8: Persist Mount in /etc/fstab

ابتدا UUID را پیدا کنید:

```bash
sudo blkid /dev/vg_data/lv_data
```

سپس:

```bash
sudo vim /etc/fstab
```

برای ext4:

```text
UUID=<filesystem_uuid> /data ext4 defaults 0 2
```

برای XFS:

```text
UUID=<filesystem_uuid> /data xfs defaults 0 0
```

بعد از تغییر `/etc/fstab` حتماً تست کنید:

```bash
sudo mount -a
```

و سپس:

```bash
df -Th /data
```

---

# 2. Scenario A: Expanding an Existing Disk

## Scale-Up

این سناریو زمانی اتفاق می‌افتد که دیسک موجود در Hypervisor بزرگ‌تر شده است.

```bash
Hypervisor
   │
   │ Increase Disk Size
   ▼
Existing Disk
/dev/sdX
   │
   ▼
Kernel Rescan
   │
   ▼
Partition / Raw Disk
   │
   ├───────────────┐
   ▼               ▼
Partitioned       Raw Disk
   │               │
   ▼               ▼
growpart          pvresize
   │               │
   ▼               │
pvresize ◄─────────┘
   │
   ▼
VG Free Space
   │
   ▼
lvextend -r
   │
   ▼
Filesystem Expanded
```

---

# Step 0: Discovery & Layer Auditing

همیشه قبل از تغییر Storage Stack، وضعیت تمام لایه‌ها را بررسی و ثبت کنید.

### Block Devices

```bash
lsblk -f
```

جزئیات بیشتر:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

### Partition Layout

```bash
sudo fdisk -l /dev/sda
sudo fdisk -l /dev/sdb
sudo fdisk -l /dev/sdc
```

یا برای یک دیسک مشخص:

```bash
sudo parted /dev/sdX print
```

### LVM Physical Volumes

```bash
sudo pvs -o pv_name,vg_name,pv_size,pv_free
```

### Volume Groups

```bash
sudo vgs -o vg_name,pv_count,lv_count,vg_size,vg_free
```

### Logical Volumes

```bash
sudo lvs -o lv_name,vg_name,lv_size,lv_attr
```

### Filesystem Usage

```bash
df -Th
```

### ⭐ مهم

قبل از ادامه باید مشخص کنیم:

```text
Disk
 │
 ├── Partitioned?
 │      │
 │      ├── YES → /dev/sdX1 / /dev/sdX2 / /dev/sdX3
 │      │
 │      └── NO  → /dev/sdX
 │
 └── Which Device is the LVM PV?
```

برای پیدا کردن PV دقیق:

```bash
sudo pvs
```

مثلاً:

```text
PV         VG       Fmt  Attr PSize    PFree
/dev/sda3  vgdocker lvm2 a--  <500.00g <20.00g
```

در این مثال PV واقعی:

```text
/dev/sda3
```

است، نه:

```text
/dev/sda
```

---

# Step 1: SCSI Bus Rescan

## Kernel Detection

بعد از افزایش Disk Size در:

```text
vSphere
Proxmox
KVM
Cloud Provider
```

ممکن است Kernel هنوز اندازه جدید دیسک را مشاهده نکرده باشد.

---

## Method 1: Per-Device Rescan

برای دیسک موجود:

```bash
echo 1 | sudo tee /sys/class/block/sda/device/rescan
```

یا:

```bash
echo 1 | sudo tee /sys/class/block/sdb/device/rescan
```

یا:

```bash
echo 1 | sudo tee /sys/class/block/sdc/device/rescan
```

سپس:

```bash
lsblk
```

اندازه جدید را بررسی کنید.

---

## Method 2: Full SCSI Bus Rescan

اگر Device مشخص نیست یا Rescan قبلی کافی نبود:

```bash
for host in /sys/class/scsi_host/host*/scan; do
    echo "- - -" | sudo tee "$host" > /dev/null
done
```

سپس:

```bash
lsblk
```

و:

```bash
dmesg | tail -50
```

---

# Step 2: Determine Disk Layout

بعد از Rescan مشخص کنید PV روی چه چیزی قرار دارد.

```bash
sudo pvs
```

سپس:

```bash
lsblk -f
```

دو حالت اصلی داریم:

```text
                 LVM PV
                   │
          ┌────────┴────────┐
          │                 │
          ▼                 ▼
     Partitioned          Raw Disk
          │                 │
     /dev/sda3           /dev/sdb
          │                 │
          ▼                 ▼
      growpart           pvresize
          │                 │
          ▼                 │
      pvresize ◄────────────┘
```

---

# Branch 1: Partitioned Disk

## Example

```text
/dev/sda
└── /dev/sda3
      └── LVM PV
           └── VG
                └── LV
                     └── Filesystem
```

---

## ⚠️ GPT Backup Header Notice

در دیسک‌های GPT، یک Backup GPT Header در انتهای دیسک قرار دارد.

وقتی Disk Size در Hypervisor افزایش پیدا می‌کند، Backup GPT Header قدیمی ممکن است دیگر در انتهای واقعی Disk نباشد.

در این شرایط ابزارهایی مانند `parted` ممکن است پیام مشابه زیر نشان دهند:

```text
Warning: Not all of the space available to /dev/sdX appears to be used,
you can fix the GPT to use all of the space.
```

ممکن است `parted` گزینه‌هایی مانند:

```text
Fix
Ignore
```

نمایش دهد.

⚠️ قبل از انتخاب `Fix` حتماً مطمئن شوید که Disk موردنظر همان Disk صحیح است.

---

# Step 2.1: Check Partition Table

```bash
sudo parted /dev/sda print
```

یا:

```bash
sudo parted /dev/sdb print
```

یا:

```bash
sudo parted /dev/sdc print
```

همچنین می‌توان بررسی کرد:

```bash
sudo fdisk -l /dev/sda
```

---

# Step 2.2: Expand Partition

### ⭐ Primary & Recommended Method

اگر `growpart` نصب باشد:

```bash
sudo growpart /dev/sda 3
```

یعنی:

```text
/dev/sda
    │
    └── Partition 3
         │
         ▼
       /dev/sda3
```

مثال‌های دیگر:

```bash
sudo growpart /dev/sda 1
sudo growpart /dev/sdb 1
sudo growpart /dev/sdc 1
```

⚠️ `<part_num>` باید شماره واقعی Partition باشد.

مثلاً:

```bash
growpart /dev/sda 3
```

برای:

```text
/dev/sda3
```

است.

---

## Alternative Method: parted

```bash
sudo parted /dev/sda resizepart 3 100%
```

یا:

```bash
sudo parted /dev/sdb resizepart 1 100%
```

بعد بررسی کنید:

```bash
lsblk
```

---

# Step 2.3: Update Physical Volume

بعد از بزرگ شدن Partition، باید LVM PV را نیز Resize کنیم.

مثلاً اگر PV:

```text
/dev/sda3
```

است:

```bash
sudo pvresize /dev/sda3
```

مثال‌های دیگر:

```bash
sudo pvresize /dev/sdb1
sudo pvresize /dev/sdc1
```

سپس بررسی:

```bash
sudo pvs
```

و:

```bash
sudo vgs
```

باید فضای آزاد جدید در VG قابل مشاهده باشد.

---

# Branch 2: Raw Disk

## No Partition Table

در این ساختار:

```text
/dev/sdb
   │
   └── LVM PV
        │
        └── VG
             │
             └── LV
```

هیچ Partition مانند:

```text
/dev/sdb1
```

وجود ندارد.

---

# Step 2.1: Rescan Device

```bash
echo 1 | sudo tee /sys/class/block/sdb/device/rescan
```

یا:

```bash
echo 1 | sudo tee /sys/class/block/sdc/device/rescan
```

سپس:

```bash
lsblk
```

---

# Step 2.2: Resize PV Directly

اگر PV مستقیماً روی Disk ساخته شده باشد:

```bash
sudo pvresize /dev/sdb
```

یا:

```bash
sudo pvresize /dev/sdc
```

سپس:

```bash
sudo pvs
```

و:

```bash
sudo vgs
```

---

# ⚠️ Important Raw Disk Rule

اگر PV روی:

```text
/dev/sdb
```

است:

```bash
sudo pvresize /dev/sdb
```

درست است.

اگر PV روی:

```text
/dev/sdb1
```

است:

```bash
sudo pvresize /dev/sdb1
```

درست است.

این دو را نباید با یکدیگر اشتباه گرفت.

---

# Step 3: Expand Logical Volume + Filesystem

بعد از:

```text
Disk Resize
      │
      ▼
Partition Resize
      │
      ▼
PV Resize
      │
      ▼
VG Free Space
```

حالا می‌توان فضای آزاد VG را به LV اختصاص داد.

---

## Option 1: Allocate 100% of VG Free Space

```bash
sudo lvextend -r -l +100%FREE /dev/<vg_name>/<lv_name>
```

مثال:

```bash
sudo lvextend -r -l +100%FREE /dev/vgdocker/lvdocker
```

---

## Option 2: Add Fixed Amount

مثلاً اضافه کردن 50GB:

```bash
sudo lvextend -r -L +50G /dev/<vg_name>/<lv_name>
```

مثال:

```bash
sudo lvextend -r -L +50G /dev/vgdocker/lvdocker
```

---

## Option 3: Allocate Percentage of VG Free Space

مثلاً 50 درصد فضای آزاد VG:

```bash
sudo lvextend -r -l +50%FREE /dev/<vg_name>/<lv_name>
```

⚠️ دقت کنید:

```bash
+50%FREE
```

یعنی 50 درصد از **فضای آزاد VG**.

اما:

```bash
+50%VG
```

مفهوم متفاوتی دارد و به درصدی از **کل VG** اشاره می‌کند.

برای جلوگیری از اشتباه در عملیات Production، بهتر است قبل از استفاده مقدار دقیق Free Space را با این دستور بررسی کنید:

```bash
sudo vgs
```

---

# 🔍 Verify After LV Expansion

بعد از `lvextend` حتماً بررسی کنید:

```bash
sudo lvs
```

سپس:

```bash
lsblk -f
```

و:

```bash
df -Th
```

---

# 3. Scenario B: Adding a Brand New Physical Disk

## Scale-Out

در این سناریو یک Disk کاملاً جدید به Server اضافه شده است.

```bash
New Disk
   │
   ▼
Kernel Detection
   │
   ▼
/dev/sdX
   │
   ▼
pvcreate
   │
   ▼
vgextend
   │
   ▼
VG Free Space
   │
   ▼
lvextend -r
   │
   ▼
LV + Filesystem
```

---

# Step 1: Detect New Disk

ابتدا وضعیت فعلی را بررسی کنید:

```bash
lsblk
```

یا:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

سپس:

```bash
sudo fdisk -l
```

اگر Disk جدید هنوز دیده نمی‌شود، Full SCSI Rescan:

```bash
for host in /sys/class/scsi_host/host*/scan; do
    echo "- - -" | sudo tee "$host" > /dev/null
done
```

سپس:

```bash
lsblk
```

و:

```bash
dmesg | tail -50
```

---

# ⚠️ Important

هرگز صرفاً بر اساس حدس:

```text
/dev/sdb
/dev/sdc
/dev/sdd
```

را انتخاب نکنید.

ابتدا با:

```bash
lsblk
```

Disk جدید را شناسایی کنید.

مثلاً:

```text
NAME   SIZE TYPE FSTYPE MOUNTPOINTS
sda    100G disk
├─sda1  1G part /boot
└─sda2 99G part
sdb    500G disk
```

در این مثال ممکن است:

```text
/dev/sdb
```

Disk جدید باشد.

اما همیشه باید وضعیت واقعی Server بررسی شود.

---

# Step 2: Initialize New Disk as PV

⚠️ `pvcreate` روی Disk جدید، اطلاعات موجود روی آن را برای استفاده به عنوان LVM آماده می‌کند. قبل از اجرا مطمئن شوید Disk صحیح است.

```bash
sudo pvcreate /dev/sdb
```

بررسی:

```bash
sudo pvs
```

---

# Step 3: Extend Existing Volume Group

مثلاً:

```bash
sudo vgextend vgdocker /dev/sdb
```

بررسی:

```bash
sudo vgs
```

باید مقدار:

```text
VFree
```

افزایش پیدا کرده باشد.

---

# Step 4: Expand Logical Volume + Filesystem

اگر می‌خواهیم تمام فضای آزاد جدید را به LV بدهیم:

```bash
sudo lvextend -r -l +100%FREE /dev/vgdocker/lvdocker
```

سپس:

```bash
sudo lvs
```

```bash
lsblk -f
```

```bash
df -Th
```

---

# 4. Expanding Logical Volume & Filesystem

## Common Final Step

سوییچ:

```bash
-r
```

یا:

```bash
--resizefs
```

باعث می‌شود `lvextend` علاوه بر افزایش LV، Filesystem را نیز تا حد امکان Resize کند.

```bash
sudo lvextend -r -l +100%FREE /dev/<vg_name>/<lv_name>
```

ساختار:

```text
VG Free Space
      │
      ▼
   lvextend
      │
      ├──► LV Size Increase
      │
      └──► Filesystem Resize
```

---

# Option 1: Use All Free Space

```bash
sudo lvextend -r -l +100%FREE /dev/<vg_name>/<lv_name>
```

Example:

```bash
sudo lvextend -r -l +100%FREE /dev/vgdocker/lvdocker
```

---

# Option 2: Add Fixed Size

مثلاً:

```bash
sudo lvextend -r -L +50G /dev/<vg_name>/<lv_name>
```

Example:

```bash
sudo lvextend -r -L +50G /dev/vgdocker/lvdocker
```

---

# Option 3: Add Percentage of Free Space

مثلاً:

```bash
sudo lvextend -r -l +50%FREE /dev/<vg_name>/<lv_name>
```

---

# 5. Manual Filesystem Expansion

## Fallback Mode

اگر LV را بدون `-r` افزایش داده باشید:

```bash
sudo lvextend -l +100%FREE /dev/<vg_name>/<lv_name>
```

Filesystem هنوز ممکن است Resize نشده باشد.

در این حالت ابتدا نوع Filesystem را مشخص کنید:

```bash
df -Th
```

یا:

```bash
lsblk -f
```

---

# ext4 / ext3 / ext2

برای Filesystemهای خانواده ext:

```bash
sudo resize2fs /dev/<vg_name>/<lv_name>
```

Example:

```bash
sudo resize2fs /dev/mapper/vgdocker-lvdocker
```

سپس:

```bash
df -Th
```

---

# XFS

برای XFS باید `xfs_growfs` روی **Mount Point** اجرا شود، نه روی Device.

مثلاً اگر:

```text
/dev/mapper/vgdocker-lvdocker
        │
        └──► /var/lib/docker
```

است:

```bash
sudo xfs_growfs /var/lib/docker
```

سپس:

```bash
df -Th /var/lib/docker
```

---

# 🧠 Filesystem Resize Rule

```text
Filesystem
│
├── ext2 / ext3 / ext4
│      └──► resize2fs <device>
│
└── XFS
       └──► xfs_growfs <mount_point>
```

---

# 6. Complete Decision Tree

```bash
                    ┌──────────────────────────┐
                    │ Storage Change Required  │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │      What happened?     │
                    └────────────┬────────────┘
                                 │
                 ┌───────────────┴────────────────┐
                 │                                │
                 ▼                                ▼
        Existing Disk Expanded            New Disk Added
           Scale-Up                         Scale-Out
                 │                                │
                 ▼                                ▼
              Rescan                           Rescan
                 │                                │
                 ▼                                ▼
          Identify PV                     Identify /dev/sdX
                 │                                │
          ┌──────┴──────┐                         ▼
          │             │                      pvcreate
          ▼             ▼                         │
     Partitioned      Raw Disk                    ▼
          │             │                     vgextend
          ▼             ▼                         │
      growpart       pvresize                     ▼
          │                                VG Free Space
          ▼                                      │
      pvresize                                   │
          │                                      │
          └──────────────┬───────────────────────┘
                         │
                         ▼
                  VG Free Space
                         │
                         ▼
                  lvextend -r
                         │
                         ▼
                LV + Filesystem
                         │
                         ▼
                      Verify
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        pvs             vgs             lvs
                         │
                         ▼
                      df -Th
```

---

# 7. Production Verification Checklist

بعد از هر عملیات Storage، این موارد را بررسی کنید:

```bash
lsblk -f
```

```bash
sudo pvs
```

```bash
sudo vgs
```

```bash
sudo lvs
```

```bash
df -Th
```

اگر Disk یا Partition تغییر کرده:

```bash
sudo fdisk -l /dev/<disk>
```

و در صورت نیاز:

```bash
sudo parted /dev/<disk> print
```

---

# 🚨 Critical Safety Rules

```bash
# Rule 1
# Never run pvcreate on an existing production PV.

# Rule 2
# Never assume /dev/sdb or /dev/sdc is the new disk.
# Always verify with lsblk / pvs / fdisk.

# Rule 3
# If PV is /dev/sda3:
sudo pvresize /dev/sda3

# NOT:
sudo pvresize /dev/sda

# Rule 4
# If PV is directly on /dev/sdb:
sudo pvresize /dev/sdb

# NOT:
sudo pvresize /dev/sdb1

# Rule 5
# Before growpart, verify the partition number.

# Rule 6
# Before lvextend, verify VG and LV names.

# Rule 7
# Never use destructive commands such as:
# pvcreate
# mkfs
# fdisk write
# parted mklabel
# unless you have confirmed the target device.

# Rule 8
# After every layer change, verify the next layer.
```

---

# 🧩 The Complete LVM Storage Stack

```bash
┌─────────────────────────────────────────────┐
│                 Application                 │
├─────────────────────────────────────────────┤
│                Mount Point                  │
│              /data /var/lib/docker         │
├─────────────────────────────────────────────┤
│                Filesystem                   │
│               ext4 / XFS                   │
├─────────────────────────────────────────────┤
│              Logical Volume                 │
│              /dev/<vg>/<lv>                │
├─────────────────────────────────────────────┤
│               Volume Group                 │
│                 <vg_name>                   │
├─────────────────────────────────────────────┤
│           Physical Volume (PV)             │
│          /dev/sdX  OR  /dev/sdX1           │
├─────────────────────────────────────────────┤
│            Partition Table                 │
│          GPT / MBR / None                  │
├─────────────────────────────────────────────┤
│               Block Device                 │
│                 /dev/sdX                   │
├─────────────────────────────────────────────┤
│          Virtual / Physical Disk            │
│       vSphere / Proxmox / KVM / SAN        │
└─────────────────────────────────────────────┘
```

# ⭐ Golden Rule

```bash
Storage Expansion Order

Hypervisor
    │
    ▼
Kernel / SCSI Rescan
    │
    ▼
Partition Table       ← Only if partitioned
    │
    ▼
Partition              ← Only if partitioned
    │
    ▼
PV
    │
    ▼
VG
    │
    ▼
LV
    │
    ▼
Filesystem
    │
    ▼
Mount Point
```

یعنی همیشه این ترتیب ذهنی را حفظ کن:

```bash
Disk
  → Partition
    → PV
      → VG
        → LV
          → Filesystem
            → Mount Point
```

و در **Raw Disk** لایه Partition حذف می‌شود:

```bash
Disk
  → PV
    → VG
      → LV
        → Filesystem
          → Mount Point
```
