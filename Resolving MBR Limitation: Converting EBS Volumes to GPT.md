## Resolving MBR Partition Limitation for EBS Volumes on Ubuntu EC2 Instances

This document provides a detailed step-by-step guide for resolving the issue of an MBR-partitioned EBS volume not supporting disk sizes beyond 2TB. The process involves reconfiguring the disk with a GPT partition table and ensuring the system remains bootable.

---

### **Prerequisites**
1. **AWS Access**: Ensure you have the required permissions to manage EC2 instances and EBS volumes.
2. **Backup**: Create a snapshot of the affected EBS volume for recovery in case of issues.
3. **Ubuntu Instance**: Confirm the Ubuntu version running on the affected instance.

---

### **Problem Overview**
- **MBR Partition Limitation**: Master Boot Record (MBR) partitioning supports a maximum disk size of 2TB.
- **Solution**: Convert the disk to use the GUID Partition Table (GPT) format, which supports larger disk sizes.

---

### **Steps to Resolve**

#### **Step 1: Launch a Temporary EC2 Instance**
1. Create a new EC2 instance in the same **availability zone** as the affected instance.
2. Ensure the temporary instance uses the same Ubuntu version as the original.

#### **Step 2: Stop and Detach the Affected EBS Volume**
1. Stop the affected EC2 instance.
2. Detach the root EBS volume from the instance via the AWS Management Console or CLI.

#### **Step 3: Attach the EBS Volume to the Temporary Instance**
1. Attach the volume to the temporary instance as a secondary disk (e.g., `/dev/nvme1n1`).

#### **Step 4: Convert to GPT and Reconfigure Partitions**
1. **Login** to the temporary instance using SSH.
2. Install `gdisk` if not already installed:
   ```bash
   sudo apt update && sudo apt install gdisk -y
   ```
3. Launch `gdisk` for the attached volume:
   ```bash
   sudo gdisk /dev/nvme1n1
   ```

4. Perform the following operations within `gdisk`:
   - **Enter Expert Mode**:
     ```
     Command (? for help): x
     ```
   - **Set Sector Alignment**:
     ```
     Expert command (? for help): l
     Enter the sector alignment value (1-65536, default = 2048): 1
     ```
   - **Return to Main Menu**:
     ```
     Expert command (? for help): m
     ```
   - **Create a BIOS Boot Partition**:
     ```
     Command (? for help): n
     Partition number (2-128, default 2): 128
     First sector: 34
     Last sector: 2047
     Hex code or GUID: ef02
     ```
   - **Delete Old Root Partition**:
     ```
     Command (? for help): d
     Partition number: 1
     ```
   - **Recreate Root Partition**:
     ```
     Command (? for help): n
     Partition number: 1
     First sector: 2048
     Last sector: (Leave default or max value)
     Hex code or GUID: (Leave default)
     ```
   - **Write Changes**:
     ```
     Command (? for help): w
     ```

#### **Step 5: Check File System Integrity**
Run a file system check:
```bash
sudo e2fsck -f /dev/nvme1n1p1
```

#### **Step 6: Resize the Filesystem**
Extend the filesystem to use the new volume size:
```bash
sudo resize2fs /dev/nvme1n1p1
```

#### **Step 7: Ensure Bootability**
1. Mount the root partition:
   ```bash
   sudo mount /dev/nvme1n1p1 /mnt
   sudo mount --bind /proc /mnt/proc
   sudo mount --bind /sys /mnt/sys
   sudo mount --bind /dev /mnt/dev
   sudo chroot /mnt
   ```
2. Install GRUB and configure:
   ```bash
   grub-install /dev/nvme1n1
   grub-mkdevicemap
   echo "GRUB_DISABLE_OS_PROBER=true" >> /etc/default/grub
   echo "GRUB_FORCE_PARTUUID=" >> /etc/default/grub.d/40-force-partuuid.cfg
   update-grub
   exit
   ```
3. Unmount:
   ```bash
   sudo umount -l /mnt/dev /mnt/sys /mnt/proc /mnt
   ```

#### **Step 8: Reattach the EBS Volume**
1. Detach the EBS volume from the temporary instance.
2. Attach it back to the original instance as `/dev/sda1`.

#### **Step 9: Start the Affected Instance**
1. Start the instance and verify it boots successfully.

#### **Step 10: Verify Disk Size**
Check the disk size and filesystem:
```bash
lsblk
df -h
```

---

### **Verification**
Ensure:
1. The instance boots without issues.
2. The full 4TB volume is accessible.

---

### **Recommendations**
- Use GPT for volumes larger than 2TB.

This process ensures that the affected EC2 instance can utilize the entire EBS volume size while maintaining bootability.
