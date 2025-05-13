# add-vm
A comprehensive guide to VM snapshots in QEMU/KVM environments. Covers types, best practices, use cases, CLI commands, troubleshooting, and maintenance tips. Perfect for sysadmins and virtualization enthusiasts.
<div align="center">
  <h1>Virtual Machine Snapshot</h1>
  <p><em>A Complete Guide</em></p>
</div>

[![QEMU Version](https://img.shields.io/badge/QEMU-v8.1.0-blue)](https://www.qemu.org/download/)
[![KVM](https://img.shields.io/badge/KVM-Linux_Kernel-green)](https://www.linux-kvm.org/)
[![License](https://img.shields.io/badge/License-GPL--2.0-orange)](https://www.qemu.org/license/)
[![Platform](https://img.shields.io/badge/Platform-Linux-lightgrey)](https://www.qemu.org/)
##  **What Is a Snapshot?** 

A **snapshot** is a preserved state of a virtual machine at a specific point in time. It captures the VM's memory state, disk state, and configuration settings, allowing you to return to that exact state later.

In **QEMU/KVM**, snapshots leverage the capabilities of the underlying disk image format (typically qcow2) to track changes made after the snapshot point without modifying the original data.

There are two main types of snapshots in QEMU/KVM:

- **Internal snapshots**: Stored within the qcow2 disk image file itself, containing both disk and VM state
    ##### Example to help you visualize
        Day 0:
        -You have a virtual machine with a disk: myvm.qcow2, running Ubuntu.
        -The VM is clean, with basic setup and no additional software installed.

        Day 1:
        -You create an internal snapshot named before-updates.
            -This snapshot is saved inside the myvm.qcow2 file.
            -You now have a restore point you can go back to anytime.
      
        Day 2:
        -You boot the VM.
        -You install system updates and add applications (e.g., Apache, Docker).
        -All changes are saved normally in the same file, but since you have a snapshot, you can go back to the previous state.

        Now you have two options:
          ✅ Option 1 – Restore the old state (rollback):
            You decide something went wrong during updates and want to return to how it was on Day 1.
              -Your VM will now return to the exact state it had on Day 1.
              -Any changes made on Day 2 are lost (unless saved somewhere else).

          ✅ Option 2 – Delete the snapshot (cleanup):
            You’re happy with the current state and want to remove the snapshot to save space.
              -The snapshot metadata is deleted.
              -The disk file (myvm.qcow2) continues to contain your current, updated system.

          
- **External snapshots**: Created as separate files that track only the changes made after the snapshot point, leaving the base image untouched
    ##### Example to help you visualize
        Day 0:
        - You have `myvm.qcow2` with Ubuntu installed.

        Day 1:
        - You create external snapshot: `snap.qcow2`
        - VM now uses: [myvm.qcow2 as base] + [snap.qcow2 for changes]

        Day 2:
        - You update the OS and install apps → these are saved in `snap.qcow2`

        Now:
        - ❌ If you delete `snap.qcow2` → you lose all Day 2 changes.
        - ✅ If you merge `snap.qcow2` into `myvm.qcow2` → Day 2 changes are    kept, and you can delete the snapshot safely. 
##  **What a Snapshot Is Not** 

**Snapshots are not backups!** This is a critical distinction:

- Snapshots remain dependent on the original disk image
- If the base image becomes corrupted, all snapshots become unusable
- Snapshots are designed for temporary use, not long-term preservation
- They increase the risk of data loss if used as the only recovery mechanism

Many administrators mistakenly rely on snapshots as backups, which can lead to catastrophic data loss when the underlying storage fails.

##  **Pros and Cons of Snapshots** 

### Advantages 

- **Speed**: Nearly instantaneous creation and restoration
- **Testing safety net**: Perfect for experimenting with system changes
- **Flexible rollback**: Multiple restore points for different configurations
- **Disk efficiency**: Uses sparse allocation to minimize initial space usage
- **Training environments**: Create consistent starting points for training sessions

### Disadvantages 

- **Performance impact**: Each active snapshot adds I/O overhead
- **Growing disk usage**: Snapshots expand as changes accumulate
- **Chain dependency**: Corruption in any part of the chain affects all subsequent snapshots
- **Management complexity**: Longer chains become difficult to track and maintain
- **Not crash-consistent** by default: Application data might not be in a consistent state

##  **Real-World Use Cases** 

### Good Use Cases 

- **Software testing**: Test application installations or updates
- **Security testing**: Experiment with potentially malicious software in isolation
- **Training environments**: Reset VMs to a clean state after each session
- **Development sandboxes**: Test code changes with easy rollback
- **Before system updates**: Create a quick restore point before applying patches

### Bad Use Cases 

- **Production database servers**: Critical systems need proper backups
- **Long-running production VMs**: Performance impact becomes significant
- **Long-term preservation**: Snapshots shouldn't be kept for months/years
- **Mission-critical systems**: Relying solely on snapshots is risky
- **As your only recovery strategy**: Always maintain proper backups

##  **Creating and Managing Snapshots in QEMU/KVM** 

### Using Command Line Tools

#### Creating a snapshot with virsh:

```bash
# Create an internal snapshot (disk + memory state)
virsh snapshot-create-as --domain vm_name --name "snapshot_name" --description "Description" --atomic

# Create an external disk-only snapshot
virsh snapshot-create-as --domain vm_name --name "snapshot_name" --disk-only --atomic
```

#### Creating a snapshot with qemu-img (for offline VMs):

```bash
# Create a snapshot of a qcow2 image
qemu-img snapshot -c snapshot_name /path/to/disk.qcow2
```

#### Listing snapshots:

```bash
# List all snapshots for a VM
virsh snapshot-list vm_name

# Get detailed info about a specific snapshot
virsh snapshot-info vm_name --snapshotname snapshot_name
```

#### Restoring snapshots:

```bash
# Restore VM to a snapshot
virsh snapshot-revert vm_name snapshot_name

# For external snapshots, you may need additional steps
virsh blockcommit vm_name vda --active --pivot
```

#### Deleting snapshots:

```bash
# Delete a specific snapshot
virsh snapshot-delete vm_name snapshot_name

# Delete all snapshots
virsh snapshot-list vm_name --name | xargs -I{} virsh snapshot-delete vm_name {}
```

### Using Virt-Manager GUI

#### Creating a snapshot:

1. Open Virt-Manager
2. Right-click the VM and select **Open**
3. Click the **Show virtual hardware details** button (i-icon)
4. Select **View → Snapshots** from the menu
5. Click the **+** button to create a new snapshot
6. Enter a name and description
7. Choose whether to include VM memory state
8. Click **Finish**

#### Restoring a snapshot:

1. Navigate to the Snapshots view as above
2. Select the desired snapshot
3. Click **Run** or **Revert to** button
4. Confirm the reversion

#### Deleting a snapshot:

1. Navigate to the Snapshots view
2. Select the snapshot to delete
3. Click the **Delete** button
4. Confirm deletion

##  **Snapshot Formats and Storage Behavior** 

QEMU/KVM snapshots primarily rely on the **qcow2** (QEMU Copy-On-Write version 2) disk format, which offers:

- **Copy-on-write mechanism**: Only changed blocks are written to the snapshot layer
- **Sparse file allocation**: Files only use the space they actually need
- **Chain-based layering**: Changes are tracked in layers that build upon each other

When you create a snapshot, the original qcow2 file becomes read-only, and a new overlay file tracks all subsequent changes. This creates a "chain" of dependencies:

```
base.qcow2 <- snapshot1.qcow2 <- snapshot2.qcow2 <- current_state.qcow2
```

##  **Maintenance Tips** 

- **Regularly delete unneeded snapshots**: Snapshots are not meant for long-term storage
- **Monitor disk usage** of snapshot chains with `qemu-img info --backing-chain`
- **Commit changes** to consolidate long chains: `qemu-img commit snapshot_file.qcow2`
- **Check the snapshot chain integrity** periodically
- **Never manually delete** snapshot files without proper management tools
- **Limit snapshot chain depth** to 10 or fewer for performance reasons
- **Consider committing** changes to the base image after testing is complete

## **Troubleshooting**

### Problem Description

When converting VirtualBox disk images (.vdi) to QEMU's qcow2 format, you might encounter this error when starting the VM:

```
[         0.129954] core perfctr but no constraints; unknown hardware!
(initramfs)
```

This indicates that the system failed to boot properly and dropped you into the initramfs rescue shell.

### Solution Steps

#### 1. Verify the VDI Before Conversion

Before converting, ensure your VDI disk is working properly:
- Boot it in VirtualBox to confirm it works correctly
- If it doesn't boot, repair it in VirtualBox before attempting conversion

#### 2. Proper Conversion with qemu-img

Open a terminal and run:

```bash
qemu-img convert -p -f vdi -O qcow2 01_add-vm.vdi 01_add-vm.qcow2
```

Parameters explained:
- `-p`: Shows conversion progress
- `-f vdi`: Forces source format specification
- `-O qcow2`: Sets output format
- This creates a clean file without data loss

#### 3. Create a VM with Virt-Manager (GUI Recommended)

- Launch virt-manager
- Create a new VM:
  - Select "Import an existing disk image"
  - OS: Choose "Generic Linux" (if unknown)
  - Disk: Select your converted 01_add-vm.qcow2
  - Memory: Allocate at least 2048 MB
  - CPU: 2 cores or more
  - Check "Customize configuration before installation"

#### 4. Critical Settings to Prevent the (initramfs) Error

Before finishing the VM creation, make these two vital adjustments:

**A. Disk Controller:**
- Click on the disk → Change the bus to SATA (or IDE for older systems)
- Do not use Virtio if the VM isn't configured for it

**B. CPU Configuration:**
- Go to the CPU tab → Check "Copy host CPU configuration" or select "host-passthrough"

These settings ensure hardware compatibility between your original VirtualBox VM and the new QEMU/KVM environment, preventing hardware detection issues that cause boot failures.

##  **Resources** 

### Official Documentation

- [QEMU Disk Images Documentation](https://www.qemu.org/docs/master/system/images.html)
- [Libvirt Snapshot XML Format](https://libvirt.org/formatsnapshot.html)
- [QEMU-IMG Manual Page](https://www.qemu.org/docs/master/tools/qemu-img.html)
  
