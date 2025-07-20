# VIRTUALIZATION WITH KVM/QEMU/VIRTMANAGER

## SW/HW used in tests:

**Tested OS**:
- Ubuntu 22.04 LTS
- Ubuntu 24.04 LTS

**Tested CPUs**: The following was tested using the following CPUs for the host system:
- AMD Ryzen 5 1500X
- AMD Ryzen 5 5600G
- Intel(R) Core(TM) i9-13900
- AMD Ryzen 9 7950x.

Get hardware info: `sudo lshw` or `sudo dmidecode -t 2` (for the MOBO)

**Tested GPUs**:: Tested on host systems mounting the following GPUs:
- NVIDIA GeForce GTX 1060 3GB
- NVIDIA GeForce RTX 4060
- NVIDIA GeForce RTX 4060 Ti

Get info on NVIDIA hardware: `nvidia-smi`

## Initial Host-side Operations: 

### Check if Hardware Assisted Virtualization is enabled for the CPU

	Hardware assisted virtualization: CPU technology that enables efficient full virtualization using help from hardware capabilities.
	Intel uses VMX, AMD uses SVM. If supported by your CPU, it must be enabled via the BIOS.

You can get all info about your CPU with
`cat /proc/cpuinfo`

Or just search for the flags you're interested into with
`egrep -c '(vmx|svm)' /proc/cpuinfo` --> if >0, it means that hardware assisted virtualization is enabled

### Check if KVM kernel module is ready

	KVM: Linux kernel module (low level) that transforms Linux systems into type 1 hypervisors (bare metal), enabling running of full VMs as separated processes.

`sudo apt install cpu-checker -y` 

Then run `kvm-ok`

You should get `KVM acceleration can be used`

### Install dependencies required for virtualization:

Beside KVM Linux kernel module, we'll need QEMU, Libvirt and Virt-Manager.
- QEMU is the emulator using the kvm accelerator,
- libvirt are higher-level backend APIs
- virt-manager is the graphical tool to manage VMs

Install the above via: 

`sudo apt install qemu-kvm libvirt-daemon-system virt-manager -y`

Then, check if APIs are running:

`sudo systemctl status libvirtd`

And start virt-manager for a user-friendly UX:

`sudo virt-manager`


## Create your VM:

From virt-manager (The KVM/QEMU GUI) --> Preferences --> Enable XML editing.

This allows editing of the XML files that declares the configuration of your VMs.
Then, follow the 5 configuration steps suggested by the GUI to create your VM, which are easy and straightforward.

Once done, before starting the VM, enter the VM Details (Enable "Customize configuration before install") and you will be able to access a more granular configuration for your VM:
- Overview - Chipset (Virtualized MOBO chipset. also called "Machine Type"): Select Q35 (more modern virtualized mobo chipset)
- Overview - Firmware: 
  - UEFI (UEFI is required for booting Windows OS)
  - BIOS (For Linux)
- CPU Configuration: 
  - IF YOU WANT TO EMULATE THE CPU (like a Type2 hypervisor), disable "Copy host CPU configuration (host-passthrough)": On some CPUs and OS like Windows, host-passthrough could result in an error at VM boot, so mimicking the host CPU configuration (model/family, features like the ISA, clock speed, cores and threads, cache size, security features, and capabilities like hardware-assisted virtualization support for the CPU, I/O operations..) as closely as possible may yield an error.
  So you can instead select a suitable CPU model to emulate. For instance, for AMD host CPUs you may select EPYC.
  - IF YOU WANT TO PASSTHROUGH THE CPU WHILE AVOIDING THE BOOT ERROR DESCRIBED ABOVE:
    - Enable "Copy host CPU configuration (host-passthrough)"
	- sudo -i
	- echo "options kvm ignore_msrs=1" > /etc/modprobe.d/kvm.conf

    This allows direct CPU passthrough to guests, fully exploiting usage of the processor virtualization support and making the QEMU/KVM system very close to a bare metal system (type 1 hypervisor). It has been tested using an AMD host processor.

  - Select "Manually Set CPU Topology": You can mimick the number of sockets, cores and threads of your CPU (socket will be 1 as probably you have a single processor)
  
  - Download VirtIO (.iso) paradrivers (set of drivers designed for virtual machines to improve I/O performance by leveraging paravirtualization techniques)
    
		Paravirtualization: Paravirtualization involves modifying the guest operating system to work more efficiently with the virtualization platform, making it aware that it is running in a virtualized environment.
		This awareness allows the guest OS to communicate more efficiently with the virtualization platform. The download can be performed from https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/?C=M;O=D

  - Add new devices to the VM:
    - Add hardware --> Storage --> Select or create custom storage --> Device type: CDROM - Manage.. --> Select the downloaded VirtIO.iso --> This now is SATA CDROM2

  - Remove the Tablet device (for Windows Guests)

  - Display : Select the Type "Spice Server" for better performance or VNC if you access remotely to the host to use the VMs, Listen Type: None, Enable OpenGL

  - Video: Select the "VirtIO" model , Disable 3D acceleration

  - TPM: Advanced options: select Model TIS and Version 2.0

  - Now you have SATA Disk1, SATA CDROM1, SATA CDROM2. For SATA Disk 1 (the VM storage disk) in Advanced options set Cache mode to none and Discard Mode to unmap.

  - Boot Options: Put SATA CDROM1 (the disk with the operating system) before VirtIO Disk 1 (the storage disk) for first booting. Then after the OS is installed you'll be able to boot from SATA Disk 1 (you can again change the boot order putting SATA Disk 1 as the first boot option). Just notice that in Windows 11 a first step will create the required partitions and a second reboot will ask you to install Windows on the primary partition.

  - VirtualMachine --> Redirect USB Device allows connection of USB devices to the Guest OS
  - You can also add hardware like USB devices to passthrough devices like WIFi usb pendrives.

  - Networking configuration:
    Default option for newly created VM is NAT. You can change the device model to virtio (after you installed the VirtIO driver inside the host machine) for better performance. All guest VM connected in the same NAT network can communicate (Windows VMs must have the Firewall disabled..).

    You can also connect your VM to a bridge network, setting the NIC as follows:
    - Network source : Bridge device
## check_iommu
`mokutil --sb`
that the secure boot is effectively removed.

*GPU Blacklisting inside Guest OS*: Also blacklisting some GPU drivers (nouveau) on the guest may be needed:

`sudo nano /etc/modprobe.d/blacklist-nouveau.conf`

Add following lines:

`blacklist nouveau`

`options nouveau modeset=0`

Update and reboot system:

`sudo update-initramfs -u`

`sudo reboot now`

*Tablet Device*: When working on linux guest VMs, keeping the Tablet device in the VM hardware will keep alignment between the host and guest mouse pointers.

### For Windows (7/10/11) Guest:

*Required internet connection at first boot*: At install Win11 requires network connection. To Bypass it, SHIFT(+FN)+F10 and type OOBE\BYPASSNRO. Then another rebooot will be performed but with the option of skipping the first network connection.

*Required Microsoft account at first boot*: To skip Microsoft Account access, log in with a@a.com with a random password (2024 - it doesnt work anymore, so you can simply create a burner email).

*VirtIO in Windows guests*: When you're inside the Windows VM, go to the explore resources and then to the VirtIO.iso disk.
You can install the VirtIO drivers via the .msi installer file.
Then, reboot the system, go to display options and now you can select a different screen resolution.


## GPU PASSTHROUGH

We won't do all of this if there was not a reason, right? 
Type 2 hypervisors like VirtualBox  may be able to let you create a VM pretty much the same way (well, maybe without cpu passthrough, but that alone would not be enough for us, right?)
There is something that we cannot do via type-2 hypervisors: we cannot have direct access from the guests OS to hardware resources, like a GPU.

We are going to follow this reference guide to enable our virtualization stack to spin up VMs with diurect access to the GPU
https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF , https://ubuntu.com/server/docs/gpu-virtualization-with-qemu-kvm:

### Considered Hardware configuration:
- CPU with an integrated graphic card (to be assigned to the host) --> AMD Ryzen 5 5600G / Intel(R) Core(TM) i9-13900 / Ryzen 9 7950x.
- A dedicated GPU (to be assigned to the guest OS) --> Nvidia GeForce GTX 1060 3GB / NVIDIA GeForce RTX 4060 / NVIDIA GeForce RTX 4060 Ti
- MOBO: ASUS PRIME B350M-K / HP 8AC1 / ASUS ROG STRIX B650E-F WiFi

### *First operations*

I first needed to perform some operations that were tailored to my hardware configuration. Other users will very likely need to do other stuff, if any.

- First, I needed to update the motherboard UEFI BIOS from v6002 to 6042 (to make it compatible with a newer CPU - on ASUS PRIME B350M using CPU AMD Ryzen 5 5600G )
For ASUS mobos, just download from asus.com and then copy in a formatted USB pendrive the .CAP file for v6042 (new bios/firmware) in your pendrive.
Then, put it in the USB ports in the back of the PC, reboot the system and enter the UEFI BIOS. There, for this mobo
model at least, go to Advanced Options, Tools and search for the UEFI BIOS Update section, select your drive and .CAP file and confirm 
the UEFI BIOS update.
At the end of the process I was greeted with a missing keyboard error, but substituting my wireless keyboard with a 
wired one allowed me to procede to the new UEFI BIOS and then to the OS.
Remember to re-activate the CPU virtualization from the UEFI BIOS after the mobo firmware update (For AMD CPU, Enable SVM from Advanced options)
Repeating the network connection was required.

- Then, I replaced the CPU. Required to re-activate the CPU virtualization from the UEFI BIOS after the CPU replacement (For AMD CPU, Enable SVM from Advanced options)
Repeating the network connection was also required.

- In UEFI BIOS also enabled IOMMU (on ASUS mobo couldnt find its location though, I searched it with the search bar).

- The system needs to be able to use both the integrated and the dedicated GPU: in the UEFI BIOS (ASUS)
Advanced - NB Configuration - IGFX Multi-Monitor - Enable

  BE CAREFUL THAT DOING THIS I WASNT ABLE ANYMORE TO ACCESS THE BIOS: I COULD ONLY SEE THE OS ONCE BOOTED.

  I WAS CONNECTED WITH A SINGLE MONITOR TO THE DEDICATED GPU OUTPUT, SO APPARENTELY WITH THIS NEW SETUP THE INTEGRATED GRAPHICS WAS DETECTED BUT COULDN'T BE USED SINCE THE SWITCH TO DEDICATED GPU WAS PERFORMED AFTER BOOT. SO EVERYTHING BEFORE THE BOOTING AND AFTER THE SHUTDOWN WAS LOST. TO RECOVER THIS I CONNECTED ONE MONITOR TO THE MOTHERBOARD AND ONE TO THE NVIDIA GPU, SO NOW THE INTEGRATED GRAPHICS HANDLES ONE MONITOR AND THE DEDICATED ONE IS USED ON THE SECOND MONITOR. SO NOW BIOS CAN BE ACCESSED AGAIN, AT LEAST ON THE MOBO MONITOR.

### *Check configuration of your PCI devices*

In the host OS, run

`lspci | grep VGA`

you should be able to see 2 VGA controllers, one being the integrated and one the dedicated GPU.

Executing *check_iommu.sh* script you can see the IOMMU groups  (the smallest set of physical devices that can be passed
to a virtual machine) corresponding to your PCI devices --> in my case, running

`check_iommu.sh | grep VGA`

I obtain:

`IOMMU Group 11 09:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Cezanne [1002:1638] (rev c9)`

`IOMMU Group 9 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] [10de:1c02] (rev a1)`

while running 

`lspci | grep VGA`

returns:

`01:00.0 VGA compatible controller: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] (rev a1)`

`09:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Cezanne (rev c9)`

Which means PCi device with address 01:00.0 (GeForce GTX 1060 3GB) belongs to IOMMU group 11, while PCi device with address 09:00.0 ([AMD/ATI] Cezanne) belongs to IOMMU group 9.
So my dedicated and integrated GPUs are in different groups and one of them can be dedicated to a VM without affecting the other.

**IF YOU HAVE A SINGLE IOMMU GROUP NAMED * FOR ALL DEVICES WHEN RUNNING "./check_iommu.sh", IT MEANS
THAT IOMMU IS NOT ENABLED SO CHECK THE FOLLOWING SECTION TO ENABLE IOMMU.** 

Running command:

`lspci -k | grep -A 2 VGA`

Gives you a list of the drivers used by the GPUs.

Running command

`lspci -nnk`

Gives you the same output, but for all your PCI devices.

A complete driver list can be obtained running (for nvidia gpus) running

`lsmod | grep -i nvidia`

Which in my case returns

	nvidia_uvm           1781760  0
	nvidia_drm             94208  3
	nvidia_modeset       1314816  3 nvidia_drm
	nvidia              56745984  105 nvidia_uvm,nvidia_modeset
	drm_kms_helper        249856  4 drm_display_helper,amdgpu,nvidia_drm
	drm                   700416  23 gpu_sched,drm_kms_helper,drm_display_helper,nvidia,drm_buddy,amdgpu,drm_ttm_helper,nvidia_drm,ttm
	video                  73728  3 asus_wmi,amdgpu,nvidia_modeset

Which means the drivers / kernel modules that handle the NVIDIA GPUs are : nvidia_drm, nvidia_modeset, nvidia_uvm and nvidia.


### Enable IOMMU and update GRUB:
IOMMU must be enabled on your system. To do this, you have to modify the file /etc/default/grub file running 

`sudo nano /etc/default/grub`

and adding or modifying the line with GRUB_CMDLINE_LINUX
to include amd_iommu=on or intel_iommu=on, depending on your processor. 

Then 

`sudo update-grub` 

to effectively update GRUB bootloader configuration file
(This file is typically located at /boot/grub/grub.cfg).

    EXPLANATION: When the BIOS UEFI MOBO firmware is executed, it searches for bootable devices which must have a bootloader program (in a dedicated boot sector, which may be part of a boot partition of an hard disk) which is in charge of loading the operating system kernel (which is included in the OS image that must be as well
    inside a device to be defined as "bootable"..).
    So the bootloader is a program that searches and loads in memory another program (the kernel),so it can be executed. A typical bootloader for Linux distros is GRUB.
	You can specify some command-line options that will be passed
	to the Linux kernel when a system boots using the GRUB bootloader.
	The GRUB_CMDLINE_LINUX variable allows you to configure various parameters related to the kernel and the initial ramdisk (initrd).
	IOMMU is an hardware component (phisycally present on the CPU, 
	but the motherboard must have a chipset that supports it) that provides memory and IO (Input/Output) address translation.
	Its primary function is to map virtual addresses used by devices in a system to physical addresses, allowing for efficient and secure communication between the CPU, memory, and I/O devices.
	IOMMU (Input-Output Memory Management Unit) is required for GPU passthrough in virtualization to provide the ability for a virtual machine (VM) to directly access and control a physical GPU. 
	Without IOMMU support, the hypervisor (virtualization software) would not be able to efficiently assign a dedicated GPU to a virtual machine due to certain limitations:
	- There would be no hardware support for memory address translation and mapping, which is essential for isolating the memory space of the GPU assigned to the VM: The VM and the host system would share the same physical memory address space, making it difficult to isolate and protect the memory regions used by the GPU.This lack of isolation can lead to security vulnerabilities and potential data corruption.
	- GPUs use DMA to transfer data directly to and from system memory. Without IOMMU, there would be no efficient way to control and isolate these DMA transactions. DMA transactions initiated by the GPU would not be properly translated and isolated, potentially leading to data leakage, unauthorized access to system memory, or corruption of data.
	- Without IOMMU support, the hypervisor would have difficulty in assigning a dedicated GPU directly to a VM. Device assignment would be limited to emulated or virtualized GPU solutions,which are typically slower and have reduced capabilities compared to direct GPU passthrough.

Running

`sudo dmesg | grep -i -e DMAR -e IOMMU`

You should see some stuff (having no output means your hardware doesnt support iommu.. and that's bad)


### Isolating your GPU
The most important step.
In order to assign a device to a virtual machine, this device **and all those sharing the same IOMMU group** must have their driver replaced by a stub driver (VFIO driver) in order to prevent the host machine from interacting with them.

    WHAT IS VFIO: VFIO, or Virtual Function I/O, is a Linux kernel subsystem / module that provides an IOMMU/device agnostic framework 
	for exposing direct device access to userspace applications and virtual machines,enabling them to interact with hardware devices at a low level.
	VFIO is particularly useful in virtualization environments, where it allows virtual machines to use physical devices, such as graphics cards and network interfaces, with near-native performance.
	The main purpose of VFIO is to provide secure and efficient access to hardware devices for userspace processes and virtual machines,
	while maintaining isolation between them. This is achieved through features such as IOMMU (Input/Output Memory Management Unit)
	and device assignment, which enable fine-grained control over the resources allocated to each virtual machine or process.
	VFIO is commonly used in conjunction with QEMU (Quick Emulator) and KVM (Kernel-based Virtual Machine) to enable hardware-accelerated virtualization on Linux systems.
 	By using VFIO, virtual machines can take advantage of hardware features and achieve performance levels similar to those of native applications running directly on the host system.

However, you cannot perform this driver substitution while the host system is using the GPU: due to their size and complexity, GPU drivers do not tend to support dynamic rebinding very well, 
so you cannot simply have some GPU you use on the host be transparently passed to a virtual machine without having both drivers conflict with each other.

Because of this, it is generally advised to bind VFIO drivers manually before starting the virtual machine.
So now we have to configure a GPU (or more precisely, all the devices in the GPU IOMMU group) so that VFIO drivers are bound early during the boot process, which makes said devices inactive until a virtual machine claims them or the driver is switched back.

In my case, since

`lspci -nnk`

returns

	01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] [10de:1c02] (rev a1)
	Subsystem: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] [10de:1c02]
	Kernel driver in use: nvidia
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia
	09:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Cezanne [1002:1638] (rev c9)
	Subsystem: ASUSTeK Computer Inc. Cezanne [1043:8809]
	Kernel driver in use: amdgpu
	Kernel modules: amdgpu

It means that NVIDIA GPU is currently using "nvidia" driver.
We have to replace this graphic driver with vfio-pci stub driver for GPU passthrough to work.

vfio-pci normally targets PCI devices by ID, meaning you only need to specify the IDs of the devices you intend to passthrough.
So you find the devices by IOMMU ID and assign them a vfio driver to make said device effectively unaccessible by the host. Then, you'll assign it to the guest VM.

First of all, run *check_iommu.sh* script to get the IOMMU groups and find your GPU devices:

	IOMMU Group 11 09:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Cezanne [1002:1638] (rev c9)
	IOMMU Group 9 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] [10de:1c02] (rev a1)
	IOMMU Group 9 01:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)

Now, an important consideration. We are now aware of the fact that our GPU is in the IOMMU group 9. But that's not the only device here: there is also something more in IOMMU group 9, that being the NVIDIA Audio device. That's important, because all the devices in the same IOMMU group of the GPU must be assigned vfio-pci driver as their driver. Otherwise, you'll face an error when starting your VM.
Let's also write the output of lspci -nnk for Audio device 01:00.1:

	01:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
		Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:5174]
		Kernel driver in use: snd_hda_intel
		Kernel modules: snd_hda_intel

This device is currently using snd_hda_intel driver. We must change it to vfio-pci driver too for proper isolation.

To pass your PCI device id to vfio-pci driver, you have 2 options (USE METHOD 2):

1) Modify as done above the GRUB_CMDLINE_LINUX in the file /etc/default/grub:

    `sudo nano /etc/default/grub`

	by appending the following (in my case), to isolate both the GPU and its audio controller:

	`vfio-pci.ids=10de:1c02,10de:10f1`

	then

	`sudo update-grub`

2) The IDs may be added to a modprobe conf file (create new or modify existing one)

	`sudo nano /etc/modprobe.d/vfio.conf`

	add a line:

	`options vfio-pci ids=10de:1c02,10de:10f1`
	    
		EXPLANATION: /etc/modprobe.d is a Linux directory that contains configuration files to control the automatic loading and unloading of kernel modules using the modprobe command. Vfio framework is composed by kernel modules, so you have to load them dynamically in the running Linux kernel to assign vfio drivers to your hardware devices.

	To load the kernel modules taking into account the new modifications of the modprobe configuration files:

	`sudo modprobe vfio`

	`sudo modprobe vfio_pci`

	In this way, the configuration in vfio.conf is applied during the system's initialization process, specifically when kernel modules are loaded.

	Then, you also need to update the initramfs image:

	`sudo update-initramfs -u`

		Initramfs (initial RAM filesystem) is a temporary filesystem that is loaded into memory (by the grub bootloader) during the initial boot process before the root filesystem is mounted.
		It is used to perform essential tasks required to initialize the system and prepare it for the transition to the actual root filesystem.
		The kernel then executes the initramfs to perform the necessary tasks before transitioning to the root filesystem, allowing the full operating system to start.
		In particular, it is used to load necessary kernel modules or drivers required for the operation of critical hardware components.

3) ALTERNATIVE METHODS, NOT FOR UBUNTU

	You have 3 choices:
	1) in /etc/mkinitcpio.conf, add MODULES=(... vfio_pci vfio vfio_iommu_type1 ...) and HOOKS=(... modconf ...)
    2) in /etc/booster.yaml , add modules_force_load: vfio_pci,vfio,vfio_iommu_type1
    3) in /etc/dracut.conf.d/10-vfio.conf, add force_drivers+=" vfio_pci vfio vfio_iommu_type1 "mkinitcpio, booster and dracut are tools or systems used in different Linux distributions to create and manage the initial ramdisk (initramfs)
	then reboot

After you chose the approach you prefer (I chose n° 2), reboot the system.

After you asked for loading the vfio-pci kernel module at kernel reboot,
you should perform an additional step to force this module to load 
before the graphics drivers have a chance to bind to the card.
Open file:

`sudo nano /etc/modprobe.d/vfio.conf`

And add the following lines:
	
	softdep drm pre: vfio-pci
	softdep nvidia pre: vfio-pci
	softdep nvidiafb pre: vfio-pci
	softdep nouveau pre: vfio-pci
	softdep nvidia_drm pre: vfio-pci
	softdep snd_hda_intel pre: vfio-pci

Then again

`sudo modprobe vfio`

`sudo modprobe vfio_pci`

`sudo update-initramfs -u`

**ON UBUNTU 22.04, THE METHOD THAT WORKED WAS TO USE MODPROBE WITH FILE VFIO.CONF**

Now after reboot only the display connected to the integrated graphics (so to the video output ports of the mobo) will work, in fact running:

`lspci -nnk`

returns, for the NVIDIA GPU and audio device:

	01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] [10de:1c02] (rev a1)
	Subsystem: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] [10de:1c02]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia
	01:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
	Subsystem: NVIDIA Corporation GP106 High Definition Audio Controller [10de:1c02]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel

So now all the devices in the IOMMU group of the GPU are effectively isolated since it they are using the vfio-pci driver. Now we are ready to set our VM to access the GPU.

### CONFIGURE VM TO ALLOW GPU PASSTHROUGH
Since my output for command

`lspci | grep VGA`

was

	01:00.0 VGA compatible controller: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] (rev a1)

I edited my VM .xml config file adding, inside <devices> section, the following subsections:

	<hostdev mode='subsystem' type='pci' managed='yes'>
	<driver name='vfio'/>
	<source>
		<address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
	</source>
	<address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
	</hostdev>

Now, a new PCI device will pop up in the device list of the VM. Notice that this can also be performed via the GUI (In the VM configuration --> Add Hardware --> PCI Host Device --> Select the GPU, and also its audio device, so that you are passing the whole IOMMU group you have previously isolated to the guest VM to use).


## Some Encountered Errors

**SOLVE STUCK AT BOOT WITH BLACKLISTING HASH (-13) ERROR:**
This issue can occour when installing Ubuntu (in my case, 22.04 LTS): 
The presence of uncompatible kernel graphical drivers with the Nvidia GPU will lead you stuck 
with this obnoxious error after choosing TRY/INSTALL Ubuntu in GRUB.
For this kind of uncompatibility problems, the SAFE GRAPHICS mode on GRUB must be instead selected.
Now you'll be able to boot inside Ubuntu and install it from your bootable USB drive.
BUT BEWARE: if you do not install the NVIDIA graphical drivers during the installation, 
at system reboot you'll be back to the blacklisting error.
So you have to connect to a Internet and install the Nvidia Drivers, either via the Ubuntu GUI installation wizard 
or via the terminal (open the terminal session with CTRL+ALT+T ,then ubuntu-drivers install).

**NOVEAU DEVICE ERROR:**
If you are prompted with a Noveau device not found error, you'll have to install Nvidia graphic drivers.
You can install the OS relying on the integratred graphics (enable from BIOS and switch video cable from GPU to motherboard)
and then install the drivers and switch your monitor back from motherboard to GPU.

**BUG IN VIRT-MANAGER**
When running virt-manager doing sudo virt-manager (since it wont connect to libvirt deamon otherwise)
you may notice you cannot change settings (settings changes will not be saved at reopen of virt-manager).
If you run
sudo virt-manager --debug
you may see
Failed to execute child process “dbus-launch” (No such file or directory)
To fix this
sudo apt install dbus-x11
Now you can open and reopen, keeping settings changes saved.

**CONFLICTING PCI DEVICES IDs**
Incidentally, my virbr0 network bridge is also a PCI device with the same ID of my graphic card:
That lead an error while booting the VM, so I needed to change the bus='0x01' for virbr0 to bus='0x11' (I still have network connection on the guest) in the .XML config file.

**VIDEO ISSUES**
- *Reinstall Nvidia drivers on Windows Guests* : On the guest Windows 11 I uninstalled the NVIDIA graphic drivers and reinstalled the latest ones (546.33) and I was finally able to get a graphic output to display 2 (the one connected to the GPU video output) and the Nvidia GTX 1060 is correctly detected by the guest system. Notice that in the display I needed QXL and not virtio to access the second screen.
- *Screen Refresh Rate*: Be careful the refresh rate on the OS side and the monitor itself are matched. Aim for a refresh rate on the monitor and adjust in display settings of OS.

**DUAL MONITOR AND LINUX BASED GUESTS**
You may experience a bug with dual monitor config and Linux based OS.
In particular the display server of the guest OS may not be properly configured.
If you go to /etc/X11 (assuming your display server is X11 and not wayland) you should have a xorg.conf file
This file should probably be created automatically based on you settings on Linux systems, but on Guest OS for some reason it is not creted apparentely.
This apparentely leads to no video output on monitor directly conencted to GPU.
Assuming NVIDIA GPU, you would see that nvidia-settings correctly detects the physical monitor but the os only sees a virtual monitor.
To fix this run:
nvidia-xconfig
This will automatically create a xorg.conf file in /etc/X11 and now at reboot the second monitor should be detected (only the second monitor).
You can modify this newly created file manually to fit your screen configuration.
More on this should be explored.
For single monitor config, check out looking glass project for high-performance frame buffering.

**OFFTOPIC - VMWARE WORKSTATION PLAYER HYPERVISOR**
When using VMWare Workstation Player 17 as the hypervisor,
there is a bug.. at least with Ubuntu 22.04.
Sometimes you create virtual machines to which no usb devices can be connected.
To solve this, shut down the vm and 
go the folder of the vm and search for its configuration file (.vmx).
Search for a line
usb.restrictions.defaultAllow = "FALSE"
And delete it. save the file and restart the VM. Now you can connect the usb devices.