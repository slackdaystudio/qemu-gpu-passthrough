# QEMU GPU Passthrough

This document is intended to capture my technique for smoothly binding and
unbinding GPU devices from the host system to a Virtual Machine.  It is more
of a collection of notes taken from various places, some of which are lost to me
already.

My intent is to keep this up-to-date so I have something to reference for my own
peace of mind.

## 0.0 Required Software
I'm running an Ubuntu 24.10-based system.  I needed the following;
```
sudo apt install libvirt-daemon-system libvirt-clients qemu-kvm qemu-utils virt-manager ovmf
```

## 1.0 The Hardware
This system is an older model Comet Lake CPU from Intel.  Still, it has all the
bells and whistles I need to run VMs.  This computer is also equipped with two
GPUs: the primary GPU is an AMD Radeon RX 7900 RT and a Radeon RX 6400 GPU which
I dedicate to my VMs.

I only run one VM at a time because I typically shift most of my physical
resources to my VMs, as that is where I will likely need them.

### 1.1 CPU Requirements
Check to see if you have Intels VT-d or AMDs AMD-Vi virtualization CPU
extensions enabled.  Make sure it is enabled in the BIOS first.
```
sudo dmesg | grep -E "VT-d|AMD-Vi"
```

Assuming you can see some output from that last command, you can begin by adding
the correct kernel param for your CPU model.

## 2.0 Kernel Params
For Intel, add `intel_iommu=on`, and for AMD, add `amd_iommu=on` to your kernel
params.

To make everything persist between reboots on my Ubuntu system, I needed to edit
GRUB first.

```
sudo vi /etc/default/grub
```

Find this line...
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`
```
...and add your param. 

For example, on my Intel machine, it looks like:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on"
```

Update GRUB:
```
sudo update-grub
```

### 2.1 VFIO Kernel Modules
Add the following to the end of `/etc/initramfs-tools/modules`
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
vhost-net
```

Once you have added the VFIO modules, you need to update your initramfs image to
load these on host boot.
```
sudo update-initramfs -u
```

### 2.2 Reboot
Reboot your system and we will check to see if things are working.

### 2.3 Validating Things
Run the following command to check if things came up configured properly.
```
sudo dmesg | grep -e DMAR -e IOMMU
```

The output for me that confirms things are configured is this line:
```
[    0.354991] DMAR: Intel(R) Virtualization Technology for Directed I/O
```

## 3.0 Add The Hooks
Now, we can add our QEMU hooks to bind and unbind the GPU automatically. 
QEMU has a powerful hook system in place that allows us to do all sorts of 
things, including automatically controlling hardware.

The easiest thing to do is check out the `qemu_hook_skeleton` directory and drop
it into `/etc/libvirt/hooks/`.

Let's look at a few files and go over what they do.
```
/etc/libvirt/hooks/
├── kvm.conf
├── qemu
└── qemu.d
   └── gpu-passthrough
       ├── prepare
       │   └── begin
       │       └── bind_vfio.sh
       └── release
           └── end
               └── unbind_vfio.sh
```

1. **qemu -** This is a key script for enabling our hooks to work.  It is
responsible for invoking our hooks.
2. **kvm.conf -** These are some functions and vars common to all hooks.  This
is where you define your device to pass through
3. **qemu.d** This is where we place our actual hook files
4. **qemu.d/gpu-passthrough -** This is a stub that we will be symlinking our 
VMs too

### 3.1 Finding Your IOMMU Group
To enable the binding and unbinding of your GPU, you must first determine its
iommu group.
```
find /sys/kernel/iommu_groups/ -type l -exec basename {} \;  | sort | xargs -I % lspci -nns %
```

In my case, it was these lines that were important:
```
02:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 24 [Radeon RX 6400/6500 XT/6500M] [1002:743f] (rev c7)
02:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21/23 HDMI/DP Audio Controller [1002:ab28]
```
The relevant part is at the start of the lines, those are the iommu groups for 
your hardware.  This is what you need to assign to your `kvm.conf` file on 
lines 2 and 3.

### 3.1 Symlinking
Now that everything is ready, you can start creating symlinks to define the
hooks for your domains.  Simply create a symlink to `gpu-passthrough` like so:
```
ln -s /home/libvirt/hooks/qemu.d/gpu-passthrough /home/libvirt/hooks/qemu.d/<VM_NAME>
```

Note: if you are unsure of the name of your VMs, run `sudo virsh list --all`

## 4.0 Assigning Hardware
At this point, I used Virtual Machine Manager to add the GPU device to my VMs via
the GUI.  It's a bit out of scope, but it was very easy to do.  **Add Hardware->PCI 
Host Device** and then pick the GPU and the GPU audio devices individually.

## 5.0 Wrapping Up
From then on I could get all my VMs binding and unbinding the secondary GPU in 
my machine.  Things are nice and snappy with GPU hardware in the mix now.  It 
is easy to add new VMs via symlinking, which is a nice plus to this approach.

Enjoy!


# References
https://github.com/bryansteiner/gpu-passthrough-tutorial

https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/

https://github.com/PassthroughPOST/VFIO-Tools
