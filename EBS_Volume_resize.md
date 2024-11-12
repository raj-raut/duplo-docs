Here's a comprehensive documentation on how to increase the size of an EBS volume and extend the file system in Linux.

---

### Increase EBS Volume and Extend File System on Linux

#### Overview
This guide provides step-by-step instructions for increasing the size of an Elastic Block Store (EBS) volume attached to an EC2 instance and expanding the file system to utilize the additional space. It covers both `ext4` and `xfs` file systems.

#### Prerequisites
- An EC2 instance with an attached EBS volume.
- SSH access to the EC2 instance with `sudo` privileges.

---

### Step 1: Increase the Size of the EBS Volume in AWS Console

1. In the **AWS Management Console**, navigate to **EC2 > Elastic Block Store > Volumes**.
2. Select the EBS volume you want to resize.
3. Choose **Actions > Modify Volume**.
4. Enter the new desired size and select **Modify**.
5. Wait a few minutes for AWS to complete the resizing operation.

---

### Step 2: Check the Filesystem and Available Disk Space on the Instance

1. SSH into the EC2 instance.
2. Run the following commands to check the filesystem type and the current available disk space:

   ```bash
   df -hT
   ```

   - The output will show the file system type (e.g., `ext4` or `xfs`) and the used and available space on each mounted file system.

3. Confirm the new volume size by listing the block devices:

   ```bash
   lsblk
   ```

   - Note: You should see the updated size for the EBS volume but not yet for the file system.

---

### Step 3: Extend the Partition

1. Use `growpart` to resize the root partition (typically `/dev/xvda1`):

   ```bash
   sudo growpart /dev/xvda 1
   ```

2. Run `lsblk` again to confirm that the partition has been resized:

   ```bash
   lsblk
   ```

---

### Step 4: Extend the File System

Extend the file system based on its type (`xfs` or `ext4`).

#### For `xfs` File System

1. Use `xfs_growfs` to extend the `xfs` file system:

   ```bash
   sudo xfs_growfs -d /
   ```

   - This command extends the file system to fill the partition without unmounting it.

#### For `ext4` File System

1. Use `resize2fs` to extend the `ext4` file system:

   ```bash
   sudo resize2fs /dev/xvda1
   ```

   - This command resizes the file system to match the resized partition.

---

### Step 5: Verify the Extended File System

1. Run `df -hT` to check the updated file system size:

   ```bash
   df -hT
   ```

2. You should see that the root file system now has additional space.

---

### Conclusion
Following these steps will allow you to successfully increase the size of an EBS volume and extend the file system on an EC2 instance, ensuring your applications have access to additional storage without downtime.

