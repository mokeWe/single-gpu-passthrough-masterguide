# Introduction

This guide presents (or aims to present) a clear, step-by-step approach to help intermediate Linux users set up a single GPU passthrough virtual machine. This guide is subject to change.

This guide is by **no means comprised of my work alone**. I **_strongly_** encourage you to check out the resources section at the very bottom of this page.

### Table of Contents

- [Introduction](#introduction)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
- [Host setup](#host-setup)
  - [Verify IOMMU is Enabled](#verify-iommu-is-enabled)
  - [Verify PCI Device Groups](#verify-pci-device-groups)
- [Guest setup](#guest-setup)
  - [Creating the Guest](#creating-the-guest)
  - [(Optional) Cloning the Guest](#optional-cloning-the-guest)
  - [Attaching Your PCI Devices](#attaching-your-pci-devices)
  - [(Optional) Audio Passthrough](#optional-audio-passthrough)
- [Setting up Libvirt Hooks](#setting-up-libvirt-hooks)
  - [Example Hooks](#example-hooks)
- [Conclusion](#conclusion)
  - [Resources, Credits, and Special Thanks](#resources-credits-and-special-thanks)

## Prerequisites

From here on, I will assume you have an intermediate-to-advanced understanding of your own system.

<details>
    <summary><b>Hardware requirements (click me)</b></summary>

Your hardware **must** be capable of a few things before continuing. If your CPU, GPU, or motherboard is incapable of the following, you may not be able to proceed.

- Your CPU must support hardware virtualization.
  - Intel CPUs: check [Intel's website](https://ark.intel.com/content/www/us/en/ark/search/featurefilter.html?productType=873&0_VTD=True&2_VTX=true) to see if your CPU supports VT-d.
  - AMD CPUs: All AMD CPUs released after 2011 support virtualization, except for the K10 series (2007–2013), which lacks IOMMU. If using a K10 CPU, ensure your motherboard has a dedicated IOMMU.
- Your motherboard must support IOMMU.
  - This includes both chipsets and BIOS. For a comprehensive list, check out the [Xen wiki](https://wiki.xenproject.org/wiki/VTd_HowTo), or [Wikipedia](https://en.wikipedia.org/wiki/List_of_IOMMU-supporting_hardware)
- Ideally, your guest GPU ROM supports UEFI.
  - It's likely it does, but check [this list](https://www.techpowerup.com/vgabios/) to be sure.

</details>

<details>
    <summary><b>Software requirements</b></summary>

- A desktop environment or window manager.
  - Although everything in this guide can be done from the CLI, virt-manager is fully featured and makes the process far less painful.

**Necessary packages, sorted by distribution:**

<details>
  <summary><b>Gentoo Linux</b></summary>

```bash
emerge -av qemu virt-manager libvirt ebtables dnsmasq
```

</details>

<details>
  <summary><b>Arch Linux</b></summary>

```bash
pacman -S qemu libvirt edk2-ovmf virt-manager dnsmasq ebtables
```

</details>

<details>
  <summary><b>Fedora</b></summary>

```bash
dnf install @virtualization
```

</details>

<details>
  <summary><b>Ubuntu</b></summary>

```bash
apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients bridge-utils virt-manager ovmf
```

</details>

Enable the "**libvirtd**" service using your init system of choice:

<details>
	<summary><b>SystemD</b></summary>

```bash
systemctl enable libvirtd
```

</details>

<details>
	<summary><b>runit</b></summary>

```bash
sv enable libvirtd
```

</details>

<details>
	<summary><b>OpenRC</b></summary>

```bash
rc-update add libvirtd default
```

</details>

</details>

# Host Setup

> IOMMU is a nonspecific name for Intel VT-d and AMD-Vi.
> VT-d should not be confused with VT-x. See https://en.wikipedia.org/wiki/X86_virtualization.

**Enable Intel VT-d or AMD-Vi in your BIOS**. If you can't find the setting, check your motherboard manufacturer's website; it may be labeled differently.

After enabling virtualization support in your BIOS, **set the required kernel parameter** depending on your CPU:

<details>
  <summary><b>For systems using GRUB</b></summary>

**Edit your GRUB configuration**

| # nvim /etc/default/grub                                                                                                                                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Intel**                                                                                                                                                          |
| `GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on iommu=pt ..."`                                                                                                     |
| On **AMD** systems, the kernel should automatically detect IOMMU hardware support and enable it. Continue to "[Verify IOMMU is Enabled](#verify-iommu-is-enabled)" |

**Regenerate your grub config to apply the changes.**

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

</details>

<details>
  <summary><b>For systems using systemd-boot</b></summary>

**Edit your boot entry.**

| # nvim /boot/loader/entries/\*.conf                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Intel**                                                                                                                                                          |
| `options root=UUID=...intel_iommu=on..`                                                                                                                            |
| On **AMD** systems, the kernel should automatically detect IOMMU hardware support and enable it. Continue to "[Verify IOMMU is Enabled](#verify-iommu-is-enabled)" |

</details>

Afterwards, **reboot your system for the changes to take effect**.

## Verify IOMMU is Enabled

**Check if IOMMU has been properly enabled using dmesg:**

```bash
dmesg | grep -i -e DMAR -e IOMMU
```

Look for lines indicating that IOMMU is enabled, it should look something like so:

- Intel:
  `[ 3.141592] Intel-IOMMU: enabled`
- AMD:
  `[ 3.141592] AMD-Vi: AMD IOMMUv2 loaded and initialized`

## Verify PCI Device Groups

Once IOMMU is enabled, you'll need to verify your PCI devices are properly grouped. Each device in your target PCI device's group must be passed through to the guest together.

For example, if my GPU's VGA controller is in IOMMU group 4, I must pass every other device in group 4 to the virtual machine along with it. Typically, one IOMMU group will contain all GPU-related PCI devices.

Run the following; it will output how your PCI devices are mapped to IOMMU groups. **You'll need this information later.**

```bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

# Guest setup

Add yourself to the `libvirt`, `kvm`, and `input` groups:

```bash
usermod -aG libvirt,kvm,input your-username
```

Being part of these groups will allow you to run your virtual machine(s) without root.

Make sure to log out and back in again for the changes to take effect.

## Creating the Guest

> If your guest isn't Windows 10/11, ignore Windows specific instructions.

Start `virt-manager` and create a new QEMU/KVM virtual machine. Follow the on-screen instructions up until **step four**.

- (_Windows_) Download the [Fedora VirtIO guest drivers](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md).
  - Either the "**Stable**" or "**Latest**" **_ISO_** builds will work.
- **During step four**, **uncheck** the box labeled "**Enable storage for this virtual machine**".
  - <details><summary><b>Why?</b></summary>If you opt to enable storage during this step, virt-manager will create a drive using an emulated SATA controller. This guide opts to use a VirtIO drive, as it offers superior performance.</details>
- On the final step, check the box labeled **Customize before install**.
- Click **finish** to proceed to customization.
- In the "**overview**" section of the newly opened window:
  - Change the **firmware** to **UEFI**.
  - Set the **chipset** to **Q35**.
    - <details><summary><b>Why?</b></summary>Q35 is preferable for GPU passthrough because it <b>provides a native PCIe bus</b>. i440FX <b>may</b> work for your specific use case.</details>
- In the "**CPUs**" tab, check the button labeled "**Copy host CPU configuration (host-passthrough)**"
- Staying in the "**CPUs**" tab, expand the "**Topology**". Configure your vCPU topology to your liking.
- Click "**Add Hardware**" and add a new storage device; the size is up to you. Make sure the bus type for the drive is set to "**VirtIO**."
- (_Windows_) **Add another storage device**. Select "**Select or create custom storage**"," **Manage**", then select the **VirtIO guest driver** ISO you downloaded prior.
  - Set the device type to "CDROM", and the bus type to "SATA" so that the Windows installer recognizes it.
- Click "**Begin Installation**" in the top left and install your guest OS.
  - (_Windows_) will not be able to detect the VirtIO disk we're trying to install it to, as we have not yet installed the VirtIO drivers.
    - When available, select "**Load Driver**" and load the driver contained within "**your-fedora-iso/amd64/win\***."

**Congratulations, you should now have a functioning virtual machine _without_ any PCI devices passed through.**
Shut down the VM and move on to the next section.

## (Optional) Cloning the Guest

Before continuing, you should clone your current virtual machine. In the next steps, we will be removing all devices that allow you to interface with the machine from within your desktop environment/window manager. If you end up needing to perform maintenance, it's beneficial to have a way to boot into your VM without having to unbind or rebind your GPU's drivers repeatedly.

- Right-click on your newly created VM.
- Click "**Clone...**"
- Name the cloned machine to your preference.
- **Deselect** the virtual disk associated with the original VM during the cloning process.
  - This ensures that the cloned machine does not create a new virtual disk but shares the original.

## Attaching Your PCI Devices

<details>
    <summary><b>(read me) Removing Unnecessary Virtual Integration Devices</b></summary>

Although removing these devices is **technically** optional, leaving them attached can lead to **unintended behavior** in your virtual machine. For example, if the **QXL video adapter** remains bound, the guest operating system will recognize it as the primary display. As a result, your other monitor(s) may either remain blank, be designated as non-primary, or both.

In your prime virtual machine's configuration, you should **remove** the following:

- **Every** spice related device.
  - <details><summary><b>Help! I'm getting 'error: Spice audio is not supported without spice graphics.'</b></summary>To resolve this, you will need to edit your VM's XML by hand. <b>In a terminal</b>, type <code>virsh edit <i>your-vm</i></code>. Search for every instance of <code>spice</code> and remove the associated line/section.</details>
- The **QXL video adapter**
- The **emulated mouse and keyboard**
  - **Instead**, directly pass through your **Host's mouse and keyboard**.
    - Navigate to **Add Hardware > USB Host Device > _Your mouse_**
    - Do the same for your USB keyboard.
- The **USB tablet**.

</details>

Now you may add the PCI devices you want to **pass through** to your virtual machine.

- Click "**Add Hardware**".
- Select the "**PCI Host Device**" tab.
- Within this tab, select a device within your [GPU's IOMMU group](#verify-pci-device-groups).
- Click "**Finish**"

Repeat these steps until **every** device in your GPU's IOMMU group is passed through to the Guest.

## (Optional) Audio Passthrough

Passing through audio from the guest to the host can be accomplished using most audio servers. I won't be listing all of them here because it would be very verbose and unnecessary. As mentioned above, if you're using PulseAudio, PipeWire + JACK, or Scream, check out the [Arch Wiki.](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_through_other_devices).

<details>
    <summary><b>PipeWire</b></summary>

Add the following to the `devices` section in your virtual machine's **XML**:

<table>
<tr>
<th>
$ virsh edit vm-name
</th>
</tr>

<tr>
<td>

```html
<devices>
  ...
  <audio id="1" type="pipewire" runtimeDir="/run/user/1000">
    <input name="qemuinput" />
    <output name="qemuoutput" />
  </audio>
</devices>
```

</td>
</tr>
</table>

(Please change the **runtimeDir** value from 1000 accordingly to your desktop user uid)

To resolve the common **Failed to initialize PW context** error, you can modify the qemu configuration to use your user.

<table>
<tr>
<th>
nvim /etc/libvirt/qemu.conf
</th>
</tr>

<tr>
<td>

user = "example"

</td>
</tr>
</table>

</details>

# Setting up Libvirt Hooks

**Libvirt hooks** allow custom scripts to be executed when a specific action occurs on the Host. In our case, special `start` and `stop` scripts will be triggered when our Guest machine is **started**/**stopped**. For more information, check out [this PassthroughPOST article](https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/), and the [VFIO-Tools GitHub page](https://github.com/PassthroughPOST/VFIO-Tools/).

To begin, create the required directories and files:

<details>
    <summary><b>Create directories and files</b></summary>

These commands will create the required directories and scripts and make them executable. Replace every instance of _your-vm-name_ with **your virtual machine's name**.

**Create the primary QEMU hook.**

```bash
mkdir -p /etc/libvirt/hooks && > /etc/libvirt/hooks/qemu && chmod +x $_
```

**Create your start script.**

```bash
mkdir -p /etc/libvirt/hooks/qemu.d/your-vm-name/prepare/begin && > /etc/libvirt/hooks/qemu.d/your-vm-name/prepare/begin/start.sh && chmod +x $_
```

**Create your stop script.**

```bash
mkdir -p /etc/libvirt/hooks/qemu.d/your-vm-name/release/end && > /etc/libvirt/hooks/qemu.d/your-vm-name/release/end/stop.sh && chmod +x $_
```

</details>

<details>
    <summary><b>Paste the following into the newly created 'qemu' script</b></summary>

<table>
<tr>
<th>
/etc/libvirt/hooks/qemu
</th>
</tr>

<tr>
<td>

```sh
#!/bin/bash

GUEST_NAME="$1"
HOOK_NAME="$2"
STATE_NAME="$3"

BASEDIR="$(dirname "$0")"

HOOKPATH="$BASEDIR/qemu.d/$GUEST_NAME/$HOOK_NAME/$STATE_NAME"
set -e  # If a script exits with an error, we should too.

if [ -f "$HOOKPATH" ]; then
    "$HOOKPATH" "$@"
elif [ -d "$HOOKPATH" ]; then
    while read -r file; do
        "$file" "$@"
    done <<<"$(find -L "$HOOKPATH" -maxdepth 1 -type f -executable -print)"
fi
```

</td>
</tr>
</table>

</details>

## Example hooks

> QEMU will automatically detach PCI devices passed to the guest. Typically, it is not necessary to manually bind and unbind `vfio` drivers using `modprobe` or to detach and attach PCI devices via `virsh`, as recommended in many other guides. I have found that misuse of such methods can often lead to script hangs. If those approaches are **necessary** and work for you, **that’s amazing**; however, I cannot endorse these methods for **_everyone_**.

Do **not** copy your **start/stop** scripts blindly. I **highly recommend** writing your own according to **your** needs. The most common use for hooks (_for single GPU passthrough machines, at least_) is to automatically **start** and **stop** your **display manager** and to unbind/rebind the proprietary NVIDIA drivers.

<details>
    <summary><b>Example <code>start</code> Script</b></summary>
    
<table>
<tr>
<th>
    /etc/libvirt/hooks/qemu.d/<i>your-vm-name</i>/prepare/begin/start.sh
</th>
</tr>
<tr>
<td>

```bash
#!/bin/bash

systemctl isolate multi-user.target

modprobe -r nvidia_drm
modprobe -r nvidia_modeset
modprobe -r nvidia_uvm
modprobe -r nvidia
```

</td>
</tr>
</table>

</details>

<details>
    <summary><b>Example <code>stop</code> Script</b></summary>

<table>
<tr>
<th>
    /etc/libvirt/hooks/qemu.d/<i>your-vm-name</i>/release/end/stop.sh
</th>
</tr>
<tr>
<td>

```bash
#!/bin/bash

modprobe nvidia_drm
modprobe nvidia_modeset
modprobe nvidia_uvm
modprobe nvidia

systemctl isolate graphical.target
```

</td>
</tr>
</table>

</details>

> P.S. If you're wondering what you should include in your hooks, I have one suggestion: **_less_**.

# Conclusion

Congratulations! You should now have a functional QEMU/KVM virtual machine, complete with PCI passthrough. If you think you can improve this guide **in any way**, I encourage you to **open a pull request**. For support, you're free to **open an issue here** or view the list of resources below.

## Resources, Credits, and Special Thanks

Without the assistance of the individuals listed below, wikis, and guides, **this would not have been possible**. There's a wealth of knowledge out there; **go seek it out**.

- ⭐ **The VFIO [subreddit](https://reddit.com/r/VFIO) and [wiki](https://reddit.com/r/VFIO/wiki/index)**
- ⭐ **[The VFIO Discord](https://discord.gg/f63cXwH)**
- ⭐ **[The VFIO blogspot how-to](https://vfio.blogspot.com/2015/05/vfio-gpu-how-to-series-part-1-hardware.html)**
- ⭐ **[The Arch Linux wiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)**
- [The Gentoo wiki](https://wiki.gentoo.org/wiki/GPU_passthrough_with_libvirt_qemu_kvm)
- [Libvirt documentation](https://libvirt.org/hooks.html)
- [QaidVoid's single GPU passthrough guide.](https://github.com/QaidVoid/Complete-Single-GPU-Passthrough)
  - Special thanks to this guide, as it inspired me to make this in the first place.
- [joeknock90's single GPU passthrough guide](https://github.com/joeknock90/Single-GPU-Passthrough)
- [This Google doc](https://docs.google.com/document/d/17Wh9_5HPqAx8HHk-p2bGlR0E-65TplkG18jvM98I7V8)
