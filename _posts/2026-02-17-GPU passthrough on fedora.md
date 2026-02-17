---
layout: distill
title: Painless GPU Passthrough Under Fedora
date: 2026-02-17
description: This post covers how to set-up separate boot entries for GPU passthrough on/off under Fedora Linux.
tags: code guide linux
categories: project
toc:
  - name: Who This Guide Is For
  - name: What You'll Need
  - name: How GPU Passthrough Works
  - name: Setting Up GPU Passthrough on Fedora
  - name: Maintaining the dual-mode setup
  - name: Automating It With `gpu-passthrough`
thumbnail: assets/img/blog/gpu_passthrough/gpu_passthrough_cover.png
---


# Painless GPU Passthrough Under Fedora

> **TL;DR:** If you already know what GPU passthrough is and just want a tool to manage passthrough kernels on Fedora, skip straight to the [`gpu-passthrough` script](#automating-it-with-gpu-passthrough). It automates the creation and maintenance of GRUB boot entries for VFIO GPU passthrough, saves your configuration, and handles cleanup after kernel updates. One command, 30 seconds, done.

I use Fedora Linux as my daily driver. In my research, my NVIDIA GPU is essential for training and running machine learning models on the host system. But I also use Autodesk Fusion, a cloud-based CAD/CAM/CAE platform that Autodesk only officially supports on Windows and macOS[^1]. There is no native Linux version, and despite vocal demand from the community[^2], Autodesk has shown no signs of changing course. Wine-based workarounds exist[^3], but they tend to break after updates and aren't reliable enough for production work. For a while, I got by with Autodesk's browser-based version of Fusion, but the experience is sluggish and limited compared to the native desktop application.

What I really needed was a way to run the full Windows version of Fusion with proper GPU acceleration, without giving up my Linux environment. GPU passthrough with KVM/QEMU makes this possible. It lets you run a Windows virtual machine with direct access to your physical GPU, giving you near-native graphics performance. The tricky part is that Fedora's frequent kernel updates break the passthrough configuration every time, requiring manual re-setup. To solve this, I wrote [`gpu-passthrough`](https://github.com/daniel-gomm/qemu-gpu-passthrough-fedora), a script that automates the entire process. This post explains the technical background, walks through the manual setup, and shows how the script makes ongoing maintenance painless.

**Disclaimer:** At the time of writing (02/2026), I have tested everything described below on system running Fedora 43 Workstation Edition with an AMD Ryzen 5900HX processor and an NVIDIA RTX 3050 mobile GPU. Use the instructions made here on your own risk.

## Who This Guide Is For

This guide is specifically written for people who need **two modes of operation**. Sometimes you want the GPU available to the Linux host (for ML workloads, gaming, or anything else), and sometimes you want it passed through to a Windows VM (for applications like Fusion, or certain Games that require Windows). The script I describe below manages separate GRUB boot entries that let you choose between these modes at boot time.

If you **always** want your GPU passed through to a VM and never need it on the host, you don't need separate boot entries or this script. In that case, you can simply add your IOMMU and VFIO parameters to `/etc/default/grub`, and they will be applied to every kernel automatically, including new ones after updates. The dual-entry approach only makes sense when you need the flexibility to switch.

## What You'll Need

Before diving in, make sure your hardware meets the following requirements.

You need **two GPUs**. In my case this is a laptop with an integrated AMD GPU (which drives my Fedora desktop) and a discrete NVIDIA GPU (which I either use for ML work on the host or pass through to the Windows VM). A desktop with two discrete GPUs works just as well.

Your **CPU must support hardware virtualization and IOMMU**. On Intel this means VT-x and VT-d; on AMD it means AMD-V and AMD-Vi. Most CPUs from the last decade support these features, but you'll need to make sure they are enabled in your BIOS/UEFI firmware.

Your **motherboard must support IOMMU** with reasonable IOMMU group granularity. This matters because IOMMU groups determine which devices can be passed through together (more on this below). Most modern boards handle this well, though some budget boards may group too many devices together.

You'll also need **QEMU/KVM and virt-manager** installed on your Fedora system. This [excellent guide by br0kenpixel](https://github.com/br0kenpixel/fedora-qemu-gpu-passthrough) covers the full installation and VM setup process in detail, so I won't duplicate that here. This post focuses specifically on the kernel and driver configuration that makes passthrough possible.

## How GPU Passthrough Works

To understand what the setup process is doing and why, it helps to know what's happening at a technical level. GPU passthrough relies on three layers of technology working together.

### IOMMU: Hardware-Level Device Isolation

The IOMMU (Input-Output Memory Management Unit) is the hardware foundation for passthrough. It sits between your PCI devices and system memory, translating between device-visible memory addresses and physical addresses. This translation is what makes it safe to give a device directly to a virtual machine without compromising the host's memory isolation[^5].

The IOMMU organizes PCI devices into **IOMMU groups**, which are sets of devices that share the same address space boundary. A critical rule of passthrough is that you must pass through all devices in a group together. This is why you'll often need to pass through both a GPU and its companion audio controller (used for HDMI/DisplayPort audio), since they typically share an IOMMU group[^5] [^6].

### VFIO: Binding Devices for Passthrough

VFIO (Virtual Function I/O) is the Linux kernel framework that makes PCI device passthrough possible. Normally, your GPU is claimed by a driver like `nvidia` or `nouveau` as soon as the system boots. VFIO provides an alternative driver called `vfio-pci` that you can bind to the GPU instead. When `vfio-pci` is bound to a device, the host kernel stops using that device entirely and makes it available for assignment to a virtual machine[^6] [^7].

The guest VM then sees the GPU as if it were physically connected, and the guest's native driver (for example, NVIDIA's Windows driver) takes over. This is what gives you near-native performance rather than the slow emulated graphics of a standard virtual display adapter.

### KVM/QEMU: The Virtualization Stack

KVM provides hardware-accelerated virtualization in the Linux kernel, and QEMU emulates the rest of the virtual machine's hardware. Together with libvirt and virt-manager as management frontends, they form the standard open-source virtualization stack on Linux. Once the GPU is bound to VFIO, you simply add it as a PCI host device in your VM configuration, and the guest can drive it directly[^8].

## Setting Up GPU Passthrough on Fedora

With the theory covered, here's what the actual setup process looks like on Fedora. The steps below are based on br0kenpixel's guide[^4], which provides more granular detail if you want to follow along manually.

### Step 1: Enable IOMMU in BIOS/UEFI

Reboot into your firmware settings and make sure that virtualization support and IOMMU are enabled. On Intel systems, look for "VT-d"; on AMD, look for "AMD-Vi" or "IOMMU". The exact menu location varies by motherboard manufacturer, but it's typically under an "Advanced" or "CPU Configuration" section.

### Step 2: Identify Your GPU's PCI IDs

{% include figure.liquid path="assets/img/blog/gpu_passthrough/lspci_vvnn_nvidia.png" class="img-fluid rounded z-depth-1" zoomable=true %}
**_Figure 1:_** _Result of running `lspci -vvnn | grep NVIDIA` and identifying the GPU PCI ID. My system has a only a single PCI group._

You need the vendor and device ID pairs for your GPU and any companion devices in its IOMMU group. Run `lspci -vvnn` and locate your discrete GPU. You're looking for something like this:

```
01:00.0 3D controller [0302]: NVIDIA Corporation GA107M [GeForce RTX 3050 Mobile] [10de:25a1] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation [10de:2291] (rev a1)
```

The IDs in brackets at the end of each line are what you need. In this example, the GPU is `10de:25a1` and the audio controller is `10de:2291`. Write these down as a comma-separated list (`10de:25a1,10de:2291`), since you'll need them in the next steps.

### Step 3: Create a Passthrough Kernel Boot Entry

This is the step that distinguishes the dual-mode approach from always-on passthrough. Rather than modifying your default kernel (which would blacklist the NVIDIA driver and make the GPU unavailable to the host), you create a **separate boot entry** with passthrough-specific parameters. This way, your normal kernel remains untouched and you can choose at boot time whether you want the GPU on the host or reserved for a VM.

Fedora uses `grubby` as its GRUB management tool, which lets you clone, modify, and remove individual boot entries. First, clone the current default kernel into a new entry:

```bash
DEFAULT_KERNEL=$(sudo grubby --default-kernel)
DEFAULT_INITRD=$(sudo grubby --info=$DEFAULT_KERNEL | grep -oP 'initrd="\K[^"]+' --max-count=1)

sudo grubby --grub2 --add-kernel=$DEFAULT_KERNEL \
    --initrd=$DEFAULT_INITRD --copy-default \
    --title="$(sudo grubby --default-title) [KVM GPU Passthrough]"
```

### Step 4: Set Kernel Parameters on the Passthrough Entry

The cloned entry needs several parameters. First, blacklist the NVIDIA and Nouveau drivers so they don't claim the GPU on the host:

```bash
sudo grubby --update-kernel=0 --args="rd.driver.blacklist=nouveau,nvidia,nvidiafb,nvidia-gpu \
    modprobe.blacklist=nouveau,nvidia,nvidiafb,nvidia-gpu"
```

Then enable IOMMU, disable the EFI framebuffer for the passthrough GPU, load the VFIO driver early, and bind it to your PCI IDs. Use `intel_iommu` or `amd_iommu` depending on your CPU:

```bash
# For AMD CPUs:
sudo grubby --update-kernel=0 --args="video=efifb:off amd_iommu=on \
    rd.driver.pre=vfio-pci kvm.ignore_msrs=1 vfio-pci.ids=YOUR_PCI_IDs"

# For Intel CPUs:
sudo grubby --update-kernel=0 --args="video=efifb:off intel_iommu=on \
    rd.driver.pre=vfio-pci kvm.ignore_msrs=1 vfio-pci.ids=YOUR_PCI_IDs"
```
Make sure to replace `YOUR_PCI_IDs` with your actual PCI IDs (`10de:25a1,10de:2291` in the example).

### Step 5: Configure Dracut (First-Time Only)

On first-time setup, you need to add the VFIO kernel modules to the initial ramdisk so they're available early in the boot process, before any other driver can claim the GPU:

```bash
echo 'add_drivers+=" vfio vfio_iommu_type1 vfio_pci "' | sudo tee -a /etc/dracut.conf.d/local.conf
sudo dracut -f --kver $(uname -r)
```

### Step 6: Set the Default Boot Entry

Decide whether the passthrough kernel should be your default. If you want to boot into passthrough mode by default, leave the new entry at index 0. If you'd rather keep the normal kernel as default and select passthrough manually at boot time, push the normal kernel back to position 0:

```bash
sudo grubby --set-default-index=1
```

After rebooting into the passthrough kernel, your NVIDIA GPU will no longer be visible to the host. You can now assign it to your Windows VM in virt-manager by adding it as a PCI host device.

{% include figure.liquid path="assets/img/blog/gpu_passthrough/grub_gpu_passthrough.jpg" class="img-fluid rounded z-depth-1" zoomable=true %}
**_Figure 2:_** _Selection between different boot entries in the GRUB boot menu._

## A Note on the GPU Reset Bug

Some AMD and NVIDIA GPUs suffer from what the VFIO community calls a "reset bug." The symptom is that the GPU refuses to re-initialize after a VM is shut down, meaning you cannot start the VM again without rebooting the entire host. This happens because QEMU fails to correctly reset the card's internal state when the guest releases it[^9].

The issue has historically been most common with AMD GPUs, particularly Polaris (RX 400/500 series), Vega, and some Navi cards[^10] [^11]. For AMD cards, a kernel module called `vendor-reset`[^10] provides device-specific reset procedures that can work around the bug on many affected models. Some NVIDIA GPUs can also exhibit reset issues, though they tend to be less widespread and often manifest differently, such as long delays during re-initialization rather than a complete failure[^12].

For the dual-mode workflow described in this guide, the reset bug is less of a concern in practice. Since you are rebooting to switch between passthrough and normal mode anyway, the GPU gets a clean hardware reset every time. The bug primarily affects setups where you want to start and stop VMs repeatedly without rebooting, such as server or headless configurations. Still, it is worth being aware of: if you encounter issues where your VM refuses to start a second time within the same boot, the reset bug is a likely culprit, and rebooting the host will resolve it.

## Maintaining the dual-mode setup

If the setup described above were a one-time process, the manual steps would be perfectly fine. The problem is that **Fedora ships kernel updates frequently**, often multiple times per month. Each new kernel version gets its own GRUB boot entry automatically, but your passthrough kernel is a clone of a *specific* kernel version. When a new kernel arrives, the new default kernel doesn't have your passthrough parameters, and your old passthrough entry still points at the older kernel. You may also want to clean up stale entries to keep your boot menu manageable.

This means that after every kernel update, you need to repeat steps 3 through 6. Clone the new kernel, apply the same parameters, set the default, and delete the old passthrough entry. Doing this once or twice a month gets tedious, and getting it wrong means booting into a state where either your desktop has no GPU or your VM can't find one.

This is a consequence of the dual-mode approach. If you were always passing through the GPU, your parameters in `/etc/default/grub` would carry over to new kernels automatically. The flexibility of being able to switch modes comes with the cost of per-kernel configuration, and that's exactly the cost this script eliminates.

## Automating It With `gpu-passthrough`

This is why I wrote [`gpu-passthrough`](https://github.com/daniel-gomm/qemu-gpu-passthrough-fedora), a script that reduces the entire update process to a single interactive command. It handles both initial setup and repeated updates with the same workflow.

On first run, the script asks for your CPU manufacturer (Intel or AMD) and your VFIO PCI IDs. It saves this configuration to `~/.config/gpu-passthrough/` so that on subsequent runs, it simply confirms the existing settings rather than asking again. Every time you run it, the script clones whichever kernel is currently the default (which after a system update will be the newest kernel), applies all the necessary parameters, offers to set the default boot entry, and cleans up any previously created passthrough entry. On the very first run, it also handles the dracut configuration automatically.

### Installation

Install the script with a single command:

```bash
curl -fsSL https://raw.githubusercontent.com/daniel-gomm/qemu-gpu-passthrough-fedora/refs/heads/main/install.sh | bash
source ~/.bashrc
```

### Usage

After a kernel update (or for first-time setup), just run:

```bash
gpu-passthrough
```

The script walks you through the process interactively, confirming your settings and showing you exactly what it's about to do before making any changes. The entire flow takes about 30 seconds compared to several minutes of manual `grubby` commands and the risk of typos or missed steps.

{% include figure.liquid path="assets/img/blog/gpu_passthrough/run_script.gif" class="img-fluid rounded z-depth-1" %}
**_Figure 3:_** _Example of running the `gpu-passthrough` script._

## Putting It All Together

My day-to-day workflow with this setup looks like this. Most of the time, I boot into the normal Fedora kernel, where the NVIDIA GPU is available for ML training and inference. When I need Fusion for CAD work, I reboot, select the passthrough kernel from the GRUB menu, and start my Windows VM. The NVIDIA GPU is fully available to Windows with native driver support and near-native performance. When Fedora pushes a kernel update, I run `gpu-passthrough`, confirm my settings, and I'm done.

For the full VM setup process, including QEMU installation, Windows VM creation, and optional Looking Glass configuration (which lets you view the VM's display in a window on your Linux host), I highly recommend following br0kenpixel's guide[^4]. The `gpu-passthrough` script is designed to complement that guide by automating the kernel configuration that you'd otherwise need to redo after every update.

{% include figure.liquid path="assets/img/blog/gpu_passthrough/virt_manager_gpu.png" class="img-fluid rounded z-depth-1" zoomable=true %}
**_Figure 4:_** _GPU listed as PCI host device for VM in virt-manager._

The script is open source under the MIT License and available on GitHub[^13]. Contributions and feedback are welcome!

[^1]: Autodesk, "Can Fusion be installed on Linux?" [https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Is-there-any-way-to-install-Fusion-360-in-Linux.html](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Is-there-any-way-to-install-Fusion-360-in-Linux.html)

[^2]: Autodesk Community Forums, "Web or native Linux version plans?" [https://forums.autodesk.com/t5/fusion-support-forum/web-or-native-linux-version-plans/td-p/13790425](https://forums.autodesk.com/t5/fusion-support-forum/web-or-native-linux-version-plans/td-p/13790425)

[^3]: Steve Zabka, "Autodesk Fusion 360 for Linux" (Wine-based installation scripts). [https://github.com/cryinkfly/Autodesk-Fusion-360-for-Linux](https://github.com/cryinkfly/Autodesk-Fusion-360-for-Linux)

[^4]: br0kenpixel, "Fedora QEMU GPU Passthrough Guide." [https://github.com/br0kenpixel/fedora-qemu-gpu-passthrough](https://github.com/br0kenpixel/fedora-qemu-gpu-passthrough)

[^5]: Arch Wiki, "PCI passthrough via OVMF." [https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)

[^6]: Gentoo Wiki, "GPU passthrough with virt-manager, QEMU, and KVM." [https://wiki.gentoo.org/wiki/GPU_passthrough_with_virt-manager,_QEMU,_and_KVM](https://wiki.gentoo.org/wiki/GPU_passthrough_with_virt-manager,_QEMU,_and_KVM)

[^7]: Ubuntu Server Documentation, "GPU virtualisation with QEMU/KVM." [https://documentation.ubuntu.com/server/how-to/graphics/gpu-virtualization-with-qemu-kvm/](https://documentation.ubuntu.com/server/how-to/graphics/gpu-virtualization-with-qemu-kvm/)

[^8]: Cloudrift, "Host Setup for QEMU KVM GPU Passthrough with VFIO on Linux." [https://www.cloudrift.ai/blog/host-setup-for-qemu-kvm-gpu-passthrough-with-vfio-on-linux](https://www.cloudrift.ai/blog/host-setup-for-qemu-kvm-gpu-passthrough-with-vfio-on-linux)

[^9]: jkhsjdhjs, "A collection of links regarding GPU passthrough." [https://github.com/jkhsjdhjs/gpu-passthrough](https://github.com/jkhsjdhjs/gpu-passthrough)

[^10]: Nicholas Sherlock, "Working around the AMD GPU Reset bug on Proxmox using vendor-reset." [https://www.nicksherlock.com/2020/11/working-around-the-amd-gpu-reset-bug-on-proxmox/](https://www.nicksherlock.com/2020/11/working-around-the-amd-gpu-reset-bug-on-proxmox/)

[^11]: Proxmox Forum, "AMD GPU inaccessible after VM Poweroff." [https://forum.proxmox.com/threads/amd-gpu-inaccessible-after-vm-poweroff-unable-to-change-power-state-from-d3cold-to-d0-device-inaccessible.130975/](https://forum.proxmox.com/threads/amd-gpu-inaccessible-after-vm-poweroff-unable-to-change-power-state-from-d3cold-to-d0-device-inaccessible.130975/)

[^12]: Proxmox Forum, "3 Minute Delay Starting VM with GPU Passthrough (vfio-pci reset issue)." [https://forum.proxmox.com/threads/3-minute-delay-starting-vm-with-gpu-passthrough-vfio-pci-reset-issue.174809/](https://forum.proxmox.com/threads/3-minute-delay-starting-vm-with-gpu-passthrough-vfio-pci-reset-issue.174809/)

[^13]: Daniel Gomm, "qemu-gpu-passthrough-fedora." [https://github.com/daniel-gomm/qemu-gpu-passthrough-fedora](https://github.com/daniel-gomm/qemu-gpu-passthrough-fedora)