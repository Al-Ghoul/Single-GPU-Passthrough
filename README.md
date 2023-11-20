## **Table Of Contents**
* **[IOMMU Setup](#enable--verify-iommu)**
* **[Installing Packages](#install-required-tools)**
* **[Enabling Services](#enable-required-services)**
* **[Guest Setup](#setup-guest-os)**
* **[Attching PCI Devices](#attaching-pci-devices)**
* **[Libvirt Hooks](#libvirt-hooks)**
* **[Keyboard/Mouse Passthrough](#keyboardmouse-passthrough)**


### **Enable & Verify IOMMU**
***BIOS Settings*** \
Enable ***Intel VT-d*** or ***AMD-Vi*** in BIOS settings. If these options are not present, it is likely that your hardware does not support IOMMU.

Disable ***Resizable BAR Support*** in BIOS settings. 
Cards that support Resizable BAR can cause problems with black screens following driver load if Resizable BAR is enabled in UEFI/BIOS. There doesn't seem to be a large performance penalty for disabling it, so turn it off for now until ReBAR support is available for KVM. 

***Set the kernel paramater depending on your CPU.***

<details>
  <summary><b>GRUB</b></summary>

***Edit GRUB configuration***
| /etc/default/grub |
| ----- |
| `GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on iommu=pt ..."` |
| OR |
| `GRUB_CMDLINE_LINUX_DEFAULT="... amd_iommu=on iommu=pt ..."` |

***Generate grub.cfg***
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```
</details>
### **Install required tools**
For Archlinux, check the [docs](https://wiki.archlinux.org/title/Virt-manager#Installation) to install Virt-manager and the required dependencies.

### **Enable required services**
<details>
  <summary><b>SystemD</b></summary>

  ```sh
  systemctl enable --now libvirtd
  ```
</details>

### **Setup Guest OS**

Download [virtio](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) driver (for windows vms). \
Launch ***virt-manager*** and create a new virtual machine. Select ***Customize before install*** on Final Step. \
In ***Overview*** section, set ***Chipset*** to ***Q35***, and ***Firmware*** to ***UEFI*** \
In ***CPUs*** section, set ***CPU model*** to ***host-passthrough***, and ***CPU Topology*** to whatever fits your system. \
For ***SATA*** disk of VM, set ***Disk Bus*** to ***virtio***. \
In ***NIC*** section, set ***Device Model*** to ***virtio*** \
Add Hardware > CDROM: virtio-win.iso (for windows vms) \
Now, ***Begin Installation***. Windows can't detect the ***virtio disk***, so you need to ***Load Driver*** and select ***virtio-iso/amd64/win10*** when prompted. \
After successful installation of Windows, install virtio drivers from virtio CDROM. You can then remove virtio iso.

### **Attaching PCI devices**
Remove Channel Spice, Display Spice, Video QXL, Sound ich* and other unnecessary devices. \
Now, click on ***Add Hardware***, select ***PCI Devices*** and add the PCI Host devices for your GPU's VGA and HDMI Audio.

### **Libvirt Hooks**
Libvirt hooks automate the process of running specific tasks during VM state change. \
More info at: [PassthroughPost](https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/)
Also, move the line to unload AMD kernal module below detaching devices from host. These might also apply to older AMD cards.

<details>
  <summary><b>Create Libvirt Hook</b></summary>

  ```sh
  mkdir /etc/libvirt/hooks
  touch /etc/libvirt/hooks/qemu
  chmod +x /etc/libvirt/hooks/qemu
  ```
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
MISC="${@:4}"

BASEDIR="$(dirname $0)"

HOOKPATH="$BASEDIR/qemu.d/$GUEST_NAME/$HOOK_NAME/$STATE_NAME"
set -e # If a script exits with an error, we should as well.

if [ -f "$HOOKPATH" ]; then
  eval \""$HOOKPATH"\" "$@"
elif [ -d "$HOOKPATH" ]; then
  while read file; do
    eval \""$file"\" "$@"
  done <<< "$(find -L "$HOOKPATH" -maxdepth 1 -type f -executable -print;)"
fi
  ```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Create Start Script</b></summary>
  
  ```sh
  mkdir -p /etc/libvirt/hooks/qemu.d/<machine_name>/prepare/begin
  touch /etc/libvirt/hooks/qemu.d/<machine_name>/prepare/begin/start.sh
  chmod +x /etc/libvirt/hooks/qemu.d/<machine_name>/prepare/begin/start.sh
  ```
**Note**: If you're on KDE Plasma (Wayland), you need to terminate user services alongside display-manager.

  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/<machine_name>/prepare/begin/start.sh
  </th>
  </tr>

  <tr>
  <td>

  ```sh
#!/bin/bash
set -x

# Stop display manager
systemctl stop display-manager
# systemctl --user -M YOUR_USERNAME@ stop plasma*
      
# Unbind VTconsoles: might not be needed
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# Unbind EFI Framebuffer
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

# Unload NVIDIA kernel modules
#modprobe -r nvidia_drm nvidia_modeset nvidia_uvm nvidia

Unload AMD kernel module
modprobe -r amdgpu
modprobe -r snd_hda_intel
# Detach GPU devices from host
# Use your GPU and HDMI Audio PCI host device
virsh nodedev-detach pci_0000_01_00_0
virsh nodedev-detach pci_0000_01_00_1

# Load vfio module
modprobe vfio-pci
  ```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Create Stop Script</b></summary>

  ```sh
  mkdir -p /etc/libvirt/hooks/qemu.d/<machine_name>/release/end
  touch /etc/libvirt/hooks/qemu.d/<machine_name>/release/end/stop.sh
  chmod +x /etc/libvirt/hooks/qemu.d/<machine_name>/release/end/stop.sh
  ```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/<machine_name>/release/end/stop.sh
  </th>
  </tr>

  <tr>
  <td>

  ```sh
#!/bin/bash
set -x

# Attach GPU devices to host
# Use your GPU and HDMI Audio PCI host device
virsh nodedev-reattach pci_0000_01_00_0
virsh nodedev-reattach pci_0000_01_00_1

# Unload vfio module
modprobe -r vfio-pci

# Load AMD kernel module
modprobe amdgpu
modprobe snd_hda_intel
# Rebind framebuffer to host
echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

# Load NVIDIA kernel modules
#modprobe nvidia_drm
#modprobe nvidia_modeset
#modprobe nvidia_uvm
#modprobe nvidia

# Bind VTconsoles: might not be needed
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 1 > /sys/class/vtconsole/vtcon1/bind

# Restart Display Manager
systemctl start display-manager
```

  </td>
  </tr>
  </table>
</details>

### **Keyboard/Mouse Passthrough**
In order to be able to use keyboard/mouse in the VM, you can passthrough the USB Host device.

Using USB Host Device is simple, \
***Add Hardware*** > ***USB Host Device***, add your keyboard and mouse device.

### **See Also**
> [Single GPU Passthrough Troubleshooting](https://docs.google.com/document/d/17Wh9_5HPqAx8HHk-p2bGlR0E-65TplkG18jvM98I7V8)<br/>
> [Single GPU Passthrough by joeknock90](https://github.com/joeknock90/Single-GPU-Passthrough)<br/>
> [Single GPU Passthrough by YuriAlek](https://gitlab.com/YuriAlek/vfio)<br/>
> [Single GPU Passthrough by wabulu](https://github.com/wabulu/Single-GPU-passthrough-amd-nvidia)<br/>
> [ArchLinux PCI Passthrough](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)<br/>
> [Gentoo GPU Passthrough](https://wiki.gentoo.org/wiki/GPU_passthrough_with_libvirt_qemu_kvm)<br/>

