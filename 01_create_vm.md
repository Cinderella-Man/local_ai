## Complete Step-by-Step Guide

From downloading the Ubuntu ISO in Proxmox, to setting up unattended encrypted booting (without needing to connect a monitor), to assigning a 4 TB disk and 4 GPUs to your first virtual machine.

---

## Step 1: Upload the Ubuntu ISO to Proxmox

The easiest way is to download the image directly through the Proxmox web interface using the official Ubuntu link.

1. In the left menu of Proxmox, select **local (pve)** storage.
2. Click the **ISO Images** tab.
3. Click **Download from URL**.
4. Paste the direct link to the Ubuntu Server image (for example, Ubuntu 24.04 LTS or 26.04):

```text
https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso
```

5. Click **Query URL**, then **Download**. Proxmox will automatically download the ISO into its storage.

---

## Step 2: Prepare Proxmox for GPU Passthrough (IOMMU)

Before assigning GPUs to a virtual machine, the Proxmox host must allow them to be passed through.

1. Open the **Shell** on your Proxmox node.
2. Edit the GRUB configuration:

```bash
nano /etc/default/grub
```

3. Find the line:

```text
GRUB_CMDLINE_LINUX_DEFAULT
```

Add the appropriate IOMMU parameters:

For **AMD CPUs**:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

For **Intel CPUs**:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

4. Save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`), then update GRUB:

```bash
update-grub
```

5. Add the required kernel modules by editing:

```bash
nano /etc/modules
```

Append:

```text
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

6. Update initramfs and reboot the Proxmox server:

```bash
update-initramfs -u -k all
reboot
```

---

## Step 3: Create the Virtual Machine and Attach the Physical 4 TB Disk

1. Click **Create VM** in the top-right corner.
2. **General:** Give the VM a name, for example `llama-server`. Remember the **VM ID** (e.g. `101`).
3. **OS:** Select the Ubuntu ISO you downloaded.
4. **System:**

   * **Machine:** `q35`
   * **BIOS:** `OVMF (UEFI)` (required for GPU passthrough)
   * **EFI Storage:** `local-lvm`
5. **Disks:** **Delete the default virtual disk** (trash can icon). You want to use the physical disk directly instead of a virtual disk image.
6. **CPU:** Assign the desired number of cores (for example, 8 or 16) and set **Type** to `host`.
7. **Memory:** Allocate an appropriate amount of RAM (for example, 32 GB or more).
8. **Network:** Leave the default `virtio` bridge.
9. Click **Finish** (do **not** start the VM yet).

### Pass Through the Raw 4 TB Disk

Open the Proxmox **Shell** and list the available disk IDs:

```bash
ls -l /dev/disk/by-id/
```

Find the identifier for your 4 TB disk (for example `/dev/disk/by-id/ata-WDC_WD4000...` or `/dev/disk/by-id/nvme-...`).

Then attach it to VM 101:

```bash
qm set 101 -scsi0 /dev/disk/by-id/YOUR_DISK_IDENTIFIER_HERE
```

---

## Step 4: Assign 4 GPUs to the VM

1. In Proxmox, select your VM (VM 101) and open the **Hardware** tab.
2. Click **Add → PCI Device**.
3. Select **Raw Device**.
4. Choose the first GPU from the list (for example `0000:01:00.0`).
5. Enable:

   * **All Functions**
   * **Primary GPU** (optional)
   * DISABLE **ROM-Bar**
   * **PCI-Express**
6. Repeat the process for the remaining three GPUs.

---

## Step 5: Install Ubuntu, Enable Disk Encryption, and Configure Headless Remote Unlocking

The goal is to encrypt the entire disk while still being able to unlock it remotely without connecting a monitor.

### Ubuntu Installation

1. Start the VM and open the **Console** tab in Proxmox. The built-in VNC console replaces the need for a physical monitor.
2. Go through the standard Ubuntu Server installer.
3. When you reach **Guided storage configuration**:

   * Select **Use an entire disk** (the installer should detect your physical 4 TB drive).
   * Check **Encrypt the new disk with LUKS**.
   * Enter your encryption passphrase (the master LUKS password).
4. Complete the installation and reboot the VM.
