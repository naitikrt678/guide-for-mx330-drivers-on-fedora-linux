# Comprehensive Guide: NVIDIA GeForce MX330 Driver Installation on Fedora

This guide provides the complete, end-to-end technical procedure for installing and configuring the proprietary NVIDIA driver for systems equipped with an NVIDIA GeForce MX330 dedicated GPU running Fedora.

The mainline NVIDIA driver (`v590+`) has dropped support for Pascal- and Maxwell-architecture chips (including the GP108M chip used in the MX330). To make this hardware function correctly under Linux, you must deploy the compatible legacy `580xx` driver branch, deactivate the open-source driver conflicts, and adjust firmware and display management parameters.

---

## Phase 1: Hardware Firmware (BIOS/UEFI) Configuration

Before installing software packages, the motherboard firmware must be configured to allow the Linux kernel to interface properly with the discrete PCIe graphics hardware.

1. **Shut Down the Computer Completely.** Turn it on and immediately tap the designated setup utility key repeatedly to enter the BIOS/UEFI configuration screen (commonly `F2` for Acer/ASUS, `F10` for HP, or `F1` / `F2` for Lenovo/Dell).
2. **Set a Supervisor Password (If Options Are Greyed Out).** Some manufacturers, particularly Acer, lock the Secure Boot toggle by default. If it is unavailable, navigate to the **Security** tab, select **Set Supervisor Password**, and create a temporary passcode.
3. **Disable Secure Boot.** Navigate to the **Boot** or **Security** tab, locate **Secure Boot**, and switch it to **Disabled**. *Unsigned or legacy proprietary third-party modules will fail to load if Secure Boot is active.*
4. **Disable Fast Boot.** Locate **Fast Boot** (typically under the **Boot** or **Main** tab) and switch it to **Disabled**. This ensures the kernel explicitly initializes all hardware on cold boot rather than bypassing setup layers using a cached state.
5. **Ensure Controller Mode is AHCI.** If present under the storage or system configuration tab, confirm the SATA or storage controller is set to **AHCI** mode rather than RAID/VMD configurations which can alter device detection layouts on Linux.
6. **Save and Exit.** Press **`F10`**, confirm the changes, and boot into Fedora.

---

## Phase 2: Environment Clean-up & Target Repositories

Any existing, partially downloaded, or incorrect mainline driver components must be completely removed from the package database to avoid conflicts during the build sequence.

### Step 1: Purge Incompatible Driver Packages

Remove any active or broken upstream NVIDIA software files while keeping the basic system-level firmware hooks intact:

```bash
sudo dnf remove "*nvidia*" --exclude nvidia-gpu-firmware

```

### Step 2: Enable RPM Fusion Repository Channels

Fedora does not package proprietary binaries natively due to licensing rules. Add the external RPM Fusion free and nonfree repositories to your distribution package manager:

```bash
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

```

### Step 3: Refresh the Local Package Cache

Synchronize the newly added software sources:

```bash
sudo dnf makecache --refresh

```

---

## Phase 3: Driver Acquisition & Native Compilation

Because proprietary modules cannot be distributed built-in to the Linux kernel, the source files must be compiled locally into binary modules mapped to your exact active kernel edition.

### Step 4: Install the 580xx Legacy Driver Stack

Deploy the specific legacy branch tracking the Pascal graphics architecture, along with matching development headers and CUDA runtime environments:

```bash
sudo dnf install xorg-x11-drv-nvidia-580xx akmod-nvidia-580xx xorg-x11-drv-nvidia-580xx-cuda kernel-devel

```

### Step 5: Trigger Manual Binary Module Build

Execute the kernel module build tool directly to force the compilation process to run immediately in the foreground:

```bash
sudo akmods --force

```

*Note: This operation compiles thousands of lines of C source code and uses maximum CPU resource limits. The user interface or mouse cursor may lock up or freeze entirely for 5 to 10 minutes depending on your processor architecture. Allow the processor to complete this work without interruption.*

### Step 6: Verify Successful Compilation

Query the kernel storage tree to confirm the binary object has been successfully generated and recognized:

```bash
modinfo -F version nvidia

```

**Expected Verified Output:** A version string beginning with `580.xx.xx`. Do not proceed if it returns a "module not found" error.

---

## Phase 4: Mitigating Driver & Display Manager Conflicts

The open-source native graphics driver (`nouveau`) and Fedora's default Wayland protocol can block the proprietary driver from taking control of the hardware layer during init routines.

### Step 7: Blacklist the Nouveau Driver

Generate a modprobe configuration file to explicitly block the kernel from loading the built-in open-source driver at boot:

```bash
echo "blacklist nouveau" | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf

```

### Step 8: Force GDM Display Manager to Use X11

Wayland handling for hybrid multi-GPU setups often skips loading the dGPU module dynamically when it defaults to power-saving parameters. Force the login shell to evaluate graphics under the X11 platform standard:

```bash
sudo sed -i 's/#WaylandEnable=false/WaylandEnable=false/' /etc/gdm/custom.conf

```

---

## Phase 5: Initializing the Device Boot Images

The final binaries and rule modifications must be embedded directly into the core filesystem image that the machine references before mounting the root storage drive.

### Step 9: Rebuild the Initramfs Filesystem

Force the system's `dracut` infrastructure to build a fresh initialization ramdisk that incorporates the new configuration parameters across all local kernel deployments:

```bash
sudo dracut --force --regenerate-all

```

### Step 10: Cycle System Power State

Perform a standard reboot to load the refreshed configuration layers:

```bash
sudo reboot

```

---

## Phase 6: Hardware Verification

Once the machine reboots and you sign back into your user account, execute the hardware validation check.

### Step 11: Poll the NVIDIA System Management Interface

Run the utility to verify direct connection with the driver:

```bash
nvidia-smi

```

#### Verification Criteria

A successful implementation will populate an ASCII interface displaying:

1. **Driver Version / CUDA Version:** `580.xx.xx` / `13.x`
2. **Product Name:** `NVIDIA GeForce MX330`
3. **Processes:** Your display server or active desktop shell environment (e.g., `/usr/bin/gnome-shell` or `Xorg`) registered inside the memory table, indicating active VRAM allocation.
