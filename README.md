# Comprehensive Guide: NVIDIA GeForce MX330 Driver Installation on Fedora 44

This document outlines the end-to-end technical procedure for configuring the proprietary NVIDIA legacy driver (`580xx` branch) on an Acer Aspire 3 (Intel 10th Gen Core i5 + Pascal-based NVIDIA MX330 GPU) running Fedora 44.

The mainline NVIDIA driver (`v590+`) drops support for Pascal/Maxwell architectures (GP108M chip found in the MX330). This manual enforces deployment of the compatible legacy branch while correcting hardware/software conflicts native to Acer firmware and Fedora's default display layers.

---

## Prerequisites & Firmware (BIOS) Configuration

Before running software installation commands, the host system firmware must be adjusted to allow the Linux kernel to interface cleanly with the discrete PCIe hardware components.

### 1. Unlock Hidden Acer BIOS Controls

Acer motherboards restrict Secure Boot modifications until an administrative lock is manually initialized.

1. Power down the system completely.
2. Initialize power and tap `F2` continuously to load the UEFI utility.
3. Use directional keys to navigate to the **Security** tab.
4. Highlight **Set Supervisor Password**, press `Enter`, and set a temporary alphanumeric passcode (e.g., `1234`).
5. Move to the **Boot** tab. The **Secure Boot** parameter will switch from greyed-out to active. Toggle it to **Disabled**.
6. Locate **Fast Boot** (found under the **Boot** or **Main** tab) and toggle it to **Disabled**. *This stops the system from using a cached state that skips runtime kernel module probing.*
7. Move to the **Main** tab and press `Ctrl + S`. Ensure SATA controller properties are strictly set to **AHCI**.
8. Press `F10` to commit modifications and reboot into the operating system.

---

## Phase 1: Environment Clean-up & Target Repositories

Any existing or partially installed mainline drivers must be wiped to avoid module symbol collision during compiling.

### Step 1: Purge Mainline Driver Tree

Remove standard upstream NVIDIA packages while leaving system-level vendor firmware hooks untouched:

```bash
sudo dnf remove "*nvidia*" --exclude nvidia-gpu-firmware

```

### Step 2: Enable RPM Fusion Repository Channels

Fedora does not ship proprietary binaries natively. Ensure the nonfree repository metadata layers are added:

```bash
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

```

### Step 3: Sync Package Metadata Cache

```bash
sudo dnf makecache --refresh

```

---

## Phase 2: Driver Acquisition & Native Compilation

### Step 4: Install the 580xx Legacy Stack

Deploy the specific legacy driver package branch tracking the Pascal architecture, alongside the required C header generation tools and CUDA framework layers:

```bash
sudo dnf install xorg-x11-drv-nvidia-580xx akmod-nvidia-580xx xorg-x11-drv-nvidia-580xx-cuda kernel-devel

```

### Step 5: Force Kernel Module Binary Generation

Because proprietary modules are built locally, compile the source code down into a raw binary interface manually.

```bash
sudo akmods --force

```

*Note: This utilizes maximum CPU resource blocks. The graphical display layer may freeze temporarily for 5–10 minutes while processing complex C headers. Do not interrupt this block.*

### Step 6: Validate Binary Availability

Confirm the compilation layer completed successfully by pulling the module version file straight from the kernel storage tree:

```bash
modinfo -F version nvidia

```

**Expected verified output:** `580.159.03` (or equivalent `580.xx` string).

---

## Phase 3: Mitigating Driver & Display Manager Blockers

The open-source kernel graphics module (`nouveau`) and Fedora's default Wayland protocol stack will preemptively lock the GPU bus unless explicitly overridden.

### Step 7: Blacklist the Nouveau Driver Module

Generate a hardware rule descriptor preventing the kernel from auto-loading open-source graphics configurations:

```bash
echo "blacklist nouveau" | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf

```

### Step 8: Force GDM Display Manager to X11

Wayland display routing often fails to safely bind hybrid multi-GPU pipelines on 10th-generation architecture. Fall back to standard X11 infrastructure parameters at the login environment level:

```bash
sudo sed -i 's/#WaylandEnable=false/WaylandEnable=false/' /etc/gdm/custom.conf

```

---

## Phase 4: Initializing the Device Boot Images

The generated driver binaries and module blacklists must be baked directly into the primary boot phase configurations.

### Step 9: Rebuild Initramfs Filesystem Structures

Force `dracut` to sweep all local kernel revisions and construct a completely updated initial ramdisk configuration embedding the new parameters:

```bash
sudo dracut --force --regenerate-all

```

### Step 10: Cycle System Execution States

Execute a system restart sequence to drop existing temporary filesystems and initialize the brand new driver structure:

```bash
sudo reboot

```

---

## Phase 5: Hardware Verification

Once the system establishes a new desktop session, verify the pipeline is fully functional.

### Step 11: Poll the NVIDIA System Management Interface

Interrogate the GPU driver communication layer:

```bash
nvidia-smi

```

#### Verification Matrix

A successful implementation will populate a structured dashboard showing:

1. **Driver Version / CUDA Version**: `580.xx.xx` / `13.x`
2. **Product Name**: `NVIDIA GeForce MX330`
3. **Processes**: Active registration of `/usr/bin/gnome-shell` (or target desktop window manager) consuming dedicated VRAM blocks.
