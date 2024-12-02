# qemu-in-vmware-howto
How to run a Win10 QEMU VM inside a VMWare ESXi VM
--------------------------------------------------

Initial problem: MS Defender does not allow Sharphound usage on domain joined assets.

Workaround: run a non-joined independent Win10 vm on a linux machine (which does not have defender on it).

Expectations:
* The linux host is already a VM --> Virtualbox / Vmware will never run in such an environment.
* No KVM support available --> Probably horrible performance

Since QEMU can act as an Emulator, no KVM is essential to run the guest OS. It could run inside a VM.

The Plan:
1. Get a Windows 10 VM (preferably vbox ova format)
2. Convert it to qemu-compatible format
3. Find the working config for the qemu vm to run
4. Run Sharphound inside the win10 vm

=====

1. Getting a Win10 VM

This one comes preinstalled, with a 90 days trial license:
  https://archive.org/details/msedge.win10.virtualbox

2. Conversion to QEMU:

First the OVA file has to be extracted with tar:
	
	tar xvf filename.ova
	
This results 2 files: a .vmdk file, which is the disk image of the VM, and a .ovf file, which is an XML describing the architecture of the VM.

The .vmdk file needs to be converted to qemu native disk format: .qcow2

	qemu-img convert -O qcow2 "MSEdge - Win10-disk002.vmdk" msedge-win10.qcow2 
	
3. Finding the working config:

This is the hard part. The qemu command need to be carefully composed based on the content of the .ovf file. Win10 requires secure boot to boot up, so an EFI bios is needed to be added as well.

After one week of googling and some trial+error, the following config proved to run the win10 vm reliably:

└─$ qemu-system-x86_64 -machine q35 -global driver=cfi.pflash01,property=secure,value=on -drive if=pflash,format=raw,unit=0,file=/usr/share/ovmf/OVMF.fd,readonly=on -m 4G -smp 2 -drive id=disk,file=msedge-win10.qcow2,if=none -device ahci,id=ahci -device ide-hd,drive=disk,bus=ahci.0 -netdev user,id=net0,net=192.168.0.0/24,dhcpstart=192.168.0.9 -device virtio-net-pci,netdev=net0 -vga virtio -cdrom virtio-win.iso -boot c --accel tcg,thread=single

Let's see what it says:

qemu-system-x86_64 : this is the qemu executable, as it will be running on a x86_64 machine

-machine q35 : this tells qemu that it has to emulate a q35 architecture, which is basically modern pc

-global driver=cfi.pflash01,property=secure,value=on -drive if=pflash,format=raw,unit=0,file=/usr/share/ovmf/OVMF.fd,readonly=on  : this two parameter add the EFI bios SPI flash device, which allows secure boot

-m 4G  : the VM will have 4 GB RAM

-smp 2 : we give the VM 2 cpu cores

-drive id=disk,file=msedge-win10.qcow2,if=none -device ahci,id=ahci -device ide-hd,drive=disk,bus=ahci.0  : with these 3 parameters we give the disk image to the vm on a virtual ahci bus, as disk device 0

-netdev user,id=net0,net=192.168.0.0/24,dhcpstart=192.168.0.9 -device virtio-net-pci,netdev=net0 : this 2 parameter adds the networking capability

-vga virtio   : we use the virtio vga driver

-cdrom virtio-win.iso  : we add this image as a cd-rom with the virtio drivers. This is basically the qemu equivalent of the virtualox guest additions and drivers. It can be downloaded https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso

-boot c : boot from the first hard driver

--accel tcg,thread=single : this is needed to avoid random crashes due to a bug in qemu's Tiny Code Generator component.

Executing the command will start the vm and windows boots up :)

[img]

4. Running SharpHound in the VM:

[img]

As expected, the performance was so poor, that it took sometimes seconds to respond for a click. But the real problem were random crashes. 

According to bug reports regarding the tcg accelerator of q35 qemu vm, I started to use single threaded CPU model for the windows VM. This made it even more slow, but -- so far -- much more stable. Ref.: https://gitlab.com/qemu-project/qemu/-/issues/1589

So, this is whay it is imprtant to add "--accel tcg,thread=single" to the command line parameters.

Finally, the - so far - stable instance look like:

	└─$ qemu-system-x86_64 -machine q35 -global driver=cfi.pflash01,property=secure,value=on -drive if=pflash,format=raw,unit=0,file=/usr/share/ovmf/OVMF.fd,readonly=on -m 8G -drive id=disk0,file=init2_msedge-win10.qcow2,if=none -device ahci,id=ahci -device ide-hd,drive=disk0,bus=ahci.0 -netdev user,id=net0,net=192.168.0.0/24,dhcpstart=192.168.0.9 -device virtio-net-pci,netdev=net0 -vga std -boot c --accel tcg,thread=single

Single threading seamingly resolved the crashes, but with the cost of even more horribly slow performance. In order to make it slightly faster, I deactivated the security features like runtime protection, etc. Remember, this is for penetartion testing, so we are not going to miss thoese features. ;)
In addition to that I also removed the preinstalled stuff and stopped some services, deactivated power management and got rid of the desktop background.

If you wonder how to run SharpHound.exe on a non domain joined computer:

	The command must be executed exactly as described in this article: https://bloodhound.readthedocs.io/en/latest/data-collection/sharphound.html#running-sharphound-from-a-non-domain-joined-system
	
	Now the collection method Group, LocalAdmin, Session, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote is running...

Update: after 4 hours of runtime the VM crashed again...
