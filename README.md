# INTRODUCTION


Hello,

I got motivated by SomeOrdinaryGamers, Mutahar, to attempt to make a simple to use VM, hidden from the guest, and quick enough to play games, so I've made this guide with my steps to make it work, this is primarily for me so I don't forget how to do this again.

I may not be doing this in the best method possible, but this is how it worked for me, so you are free to make adaptions and changes to your liking.

I also just made this for myself incase I ever want to go back to this project. I've finished it but I don't feel at home yet, so I'm going back to Windows for now.

Thanks and I hope this helped anyone who needed help.

#### Anything you do is at your own risk, this worked for me, I just want you to know that I do not guarantee this will 100% work for anyone.


# SOURCES

#### Thanks to these people, topics such as passthrough are widely available!

1. [The Passthrough post](https://passthroughpo.st)

2. [Single-GPU-Passthrough](https://github.com/joeknock90/Single-GPU-Passthrough) by joeknock90 on Github

3. [PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) by wiki.archlinux.org

4. [I Made The Greatest Windows 11 Virtual Machine...](https://www.youtube.com/watch?v=WYrTajuYhCk)  by SomeOrdinaryGamers on Youtube

5. [I Almost Lost My Virtual Machines...](https://www.youtube.com/watch?v=BUSrdUoedTo) by SomeOrdinaryGamers on Youtube

6. [Indian Man Beats VALORANT's Shady Anticheat...](https://www.youtube.com/watch?v=L1JCCdo1bG4) by SomeOrdinaryGamers on Youtube



## INSTALLATION OF ARCH LINUX


Entering the fdisk utility:
```
fdisk -l
fdisk /dev/nvme0n1 (in my case, it could also be sda)
```
```
g - for new GPT partition table
n - for new partition
t - for partition types
w - to write all changes
```

#### You would want to make a first partition as a 'System' partition, or 'EFI partition' with the last sector: ```+500M```.

#### With ```t``` change the type of the partition to ```1``` or ```EFI partition```.

#### You would also want to make a second partition, where you install arch, and leave the type as 'Linux filesystem'.


Format the first partition as FAT32:
```
mkfs.fat -F32 /dev/nvme0n1p1 
```
Format the second partition as ext4:
```
mkfs.ext4 /dev/nvme0n1p2
```
Mount the second partition:
```
mount /dev/nvme0n1p2 /mnt
```
Create a home folder:
```
mkdir /mnt/home
```
Create an fstab file:
```
mkdir /mnt/etc
genfstab -U -p /mnt >> /mnt/etc/fstab
```
Confirm everything is good:
```
cat /mnt/etc/fstab
```
Install a package group / base packages:
```
pacstrap -i /mnt base
```
To configure the installation chroot into the mounted ext4 partition:
```
arch-chroot /mnt
```
Install linux, for long-term support: ```linux-lts linux-lts-headers```
```
pacman -S linux linux-headers  
```
Install packages like nano, NetworkManager... for wifi consider: ```dialog wpa_supplicant wireless_tools netctl```
```
sudo pacman -S base-devel openssh nano networkmanager 
```
Enable NetworkManager & openssh to start when the computer starts:
```
systemctl enable NetworkManager
systemctl enable sshd
```
Generate mkinitcpio files:
```
mkinitcpio -p linux (linux-lts)
```
Time to select locales, uncomment the locales you need. I picked these: ```hr_HR.UTF-8 UTF-8 & en_us.UTF-8 UTF-8```
```
nano /etc/locale.gen
```
Generate locales:
```
locale-gen
```
Set the root password:
```
passwd
```
Create the user:
```
useradd -m -g users -G wheel "username"
```
Set the password for the user:
```
passwd "username"
```
Enable sudo for users, uncomment ```%wheel ALL=(ALL) ALL```
```
EDITOR=nano visudo
```
Get grub for UEFI:
```
pacman -S grub efibootmgr dosfstools os-prober mtools
```
Create EFI folder:
```
mkdir /boot/EFI
```
Mount EFI partition:
```
mount /dev/nvme0n1p1 /boot/EFI
```
Install grub:
```
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
```
Copy locales:
```
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```
Edit ```/etc/default/grub```:
```
nano /etc/default/grub
```

Under ```GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"``` add ```intel_iommu=on```

so it looks like this: ```GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 intel_iommu=on quiet"```.

#### You may also want to enable VD-t on intel systems in your motherboards BIOS (IOMMU won't work without it!).


Generate grub file:
```
grub-mkconfig -o /boot/grub/grub.cfg
```
Unmount drives and reboot (You should be able to boot without the installation media):
```
umount -a
reboot
```
Set the timezone, I set in my case ```Europe/Zagreb```:
```
timedatectl set-timezone Europe/Zagreb
```
Enable the synchronization of clock:
```
systemctl enable systemd-timesyncd
```
Edit vconsole.conf for keyboard kayout in console:
```
nano /etc/vconsole.conf
```
In this newly created vconsole.conf add ```KEYMAP=croat```.

Set the prefered keyboard layout for console and x11 (```croat``` & ```hr``` are for croation, use as wanted):
```
localectl set-keymap croat
localectl set-x11-keymap hr
```
Set the hostname (I used 'arch'):
```
hostnamectl set-hostname "name"
```
Edit the hosts file:
```
nano /etc/hosts
```
Add these lines at the end of the file:
```
127.0.0.1 localhost
::1 localhost
127.0.1.1 "name"
```
Install the microcode for your CPU, xorg, nvidia (nvidia-lts for linux-lts):
```
pacman -S intel-ucode xorg-server nvidia nvidia-utils
```
Install GNOME:
```
pacman -S gnome gnome-tweaks
```
Enable display manager:
```
systemctl enable gdm
```
Install yay, AUR helper:
```
sudo pacman -S git
cd /opt
sudo git clone https://aur.archlinux.org/yay-git.git
sudo chown -R "username":users ./yay-git
cd yay-git
makepkg -si
```


## SOFTWARE FOR KVM/QEMU VM


Confirm IOMMU is enabled:
```
dmesg | grep -i -e DMAR -e IOMMU
```
Indentify your gpu, make sure it is in its separe iommu group, as well as anything else you want to pass:
```
#!/bin/bash
shopt -s nullglob
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
My GPU (GTX 1060) looks like this after the previous command:
```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] [10de:1c03] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
```
#### Copy all your devices in your GPU to a seperate file to use for later.

#### If there is anything else in your IOMMU group, and it isn't PCI BRIDGE, you may want to consider ACS patching.

Install all required software for QEMU:
```
sudo pacman -S qemu libvirt edk2-ovmf virt-manager iptables-nft dnsmasq 
```
Enable and start the services:
```
sudo systemctl enable libvirtd.service virtlogd.socket
sudo systemctl start libvirtd.service virtlogd.socket
```
Activate and start the default libvirt network:
```
virsh net-autostart default
virsh net-start default
```


## DUMPING GPU VBIOS + PATCH


Get 'nvflash' from yay:
```
yay -S nvflash 
```
Get 'bless' (hex editor):
```
sudo pacman -S bless
```
Dump your vbios (you may want to do this with open-source drivers):
```
sudo nvflash --save gpu.rom
```
Using bless, load 'gpu.rom' and search for 'VIDEO' as text.

After finding 'VIDEO', a little earlier then 'VIDEO' you will find a 'U'. delete everything that comes before it.

Save as 'patch.rom'


## CREATING A VM USING VIRT-MANAGER


1. Download windows .iso from [Microsoft](https://www.microsoft.com/software-download/windows10ISO).
2. Open virt-manager.
3. Create a new VM.
4. Select local instal media.
5. If it asks for permissions, click yes.
6. Give it RAM.
7. Give it CPU.
8. You may want to use a virtual disk, or pass a disk, or even pass the entire SATA controller if it's in it's own IOMMU group.
9. Select customize before install.
10. Change chipset to ```Q35```.
11. Change firmware to ```UEFI x86_64: /usr/share/edk2-ovmf/x64/OVMF_CODE.fd```.
12. Setup topology (more advanced).
13. Enable 'SATA CDROM1' in boot options.
14. Press 'Begin Installatino' to install Windows...
15. When you finish, shut it down.


## SETTING UP A SCRIPT TO HIJACK AND RELEASE THE GPU


Get 'wget':
```
sudo pacman -S wget
```
Make a libvirt/hooks folder in '/etc/':
```
sudo mkdir -p /etc/libvirt/hooks
```
Mnstall hook manager and make it executable:
```
sudo wget 'https://raw.githubusercontent.com/PassthroughPOST/VFIO-Tools/master/libvirt_hooks/qemu' \
     -O /etc/libvirt/hooks/qemu
sudo chmod +x /etc/libvirt/hooks/qemu
```
Make a 'kvm.conf' file:
```
nano /etc/libvirt/hooks/kvm.conf
```
Add this into the 'kvm.conf' (you copied the numbers when you identified the gpu):
```
VIRSH_GPU_VIDEO=pci_0000_01_00_0
VIRSH_GPU_AUDIO=pci_0000_01_00_1
```
Get 'tree':
```
sudo pacman -S tree
```
#### You need to create scripts and folder to look like this:
```
/etc/libvirt/hooks
├── qemu <- The script that does the magic
└── qemu.d
    └── {VM Name}
        ├── prepare
        │   └── begin
        │       └── start.sh
        └── release
            └── end
                └── revert.sh
```
Create the folders and the first script to hijack the GPU:
```              
sudo mkdir /etc/libvirt/hooks/qemu.d/win10/prepare/begin
sudo nano /etc/libvirt/hooks/qemu.d/win10/prepare/begin/start.sh
```
The first, 'start.sh', script (the script is personalized towards me):
```
# debugging
set -x

# load variables we defined
source "/etc/libvirt/hooks/kvm.conf"

# stop display manager
systemctl stop gdm.service
killall gdm-x-session

# unbind VTconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# unbind EFI-framebuffer
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

# avoid race condition
sleep 2

# unload Nvidia
modprobe -r nvidia_drm
modprobe -r nvidia_modeset
modprobe -r drm_kms_helper
modprobe -r nvidia
modprobe -r i2c_nvidia_gpu
modprobe -r drm
modprobe -r nvidia_uvm

# unbind gpu
virsh nodedev-detach $VIRSH_GPU_VIDEO
virsh nodedev-detach $VIRSH_GPU_AUDIO

# load vfio
modprobe vfio
modprobe vfio_pci
modprobe vfio_iommu_type1
```
Make the script executable:
```
sudo chmod +x /etc/libvirt/hooks/qemu.d/win10/prepare/begin/start.sh
```
Make the folders and the second script to release the GPU:
```
sudo mkdir /etc/libvirt/hooks/qemu.d/win10/release/end
sudo nano /etc/libvirt/hooks/qemu.d/win10/release/end/revert.sh
```
The second, 'revert.sh', script (the script is personalized towards me):
```
# debug
set -x

# load variables
source "/etc/libvirt/hooks/kvm.conf"

# unload vfio-pci
modprobe -r vfio_pci
modprobe -r vfio_iommu_type1
modprobe -r vfio

# rebind gpu
virsh nodedev-reattach $VIRSH_GPU_VIDEO
virsh nodedev-reattach $VIRSH_GPU_AUDIO

# rebind VTconsoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# rebind nvidia x config
nvidia-xconfig --query-gpu-info > /dev/null 2>&1

# bind EFI-framebuffer
echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

#load nvidia
modprobe nvidia_drm
modprobe nvidia_modeset
modprobe drm_kms_helper
modprobe nvidia
modprobe i2c_nvidia_gpu
modprobe drm
modprobe nvidia_uvm

# restart display service
systemctl start gdm.service
```
Make the script executable:
```
sudo chmod +x /etc/libvirt/hooks/qemu.d/win10/release/end/revert.sh
```


## PASSING THE GPU TO THE VM


1. Go into virt-manager and add your GPU and it's devices (eg. digital audio) under '+Add Hardware' and 'PCI Host Device'.
2. Edit the added PCI device within virt-manager as XML.
3. Under ```</source>``` and above ```<address type='pci'...>``` add ```<rom file='/home/"user"/patch.rom'>``` (you should change this path to the 'patch.rom' path script you made earlier).
5. You should add this line to all your GPU devices.

#### Before booting the VM add following lines to your VM's XML (Some may not be needed, I added these just to be sure)(win10.xml in my case):

Inside ```<hyperv>```:
```
<hyperv>
 <relaxed state="on"/>
 <vapic state="on"/>
 <spinlocks state="on" retries="8191"/>
 <vpindex state="on"/>
 <runtime state="on"/>
 <synic state="on"/>
 <stimer state="on"/>
 <reset state="on"/>
 <vendor_id state="on" value="randomid"/>
 <frequencies state="on"/>
 <tlbflush state="on"/>
</hyperv>
```
#### After installing windows, attempt to enable Hyper-V in Windows features, it might help with anticheats (it also may make Windows not bootable, I don't know how to deal with that at the moment).

Under ```</hyperv>``` add:
```
<kvm>
 <hidden state="on"/>
</kvm>
```
Change the ```<cpu>``` section:
```
<cpu mode="host-passthrough" check="none" migratable="on">
 <topology sockets="1" dies="1" cores="3" threads="2"/> (suit your topology)
 <feature policy="disable" name="hypervisor"/>
 <feature policy="require" name="topoext"/>
</cpu>
```
Above ```<os>``` add:
```
<sysinfo type="smbios">
 <bios>
  <entry name="vendor">American Megatrends Inc.</entry>
  <entry name="version">1301</entry>
  <entry name="date">03/14/2018</entry>
  <entry name="release">5.12</entry>
 </bios>
 <system>
  <entry name="manufacturer">ASUSTeK COMPUTER INC.</entry>
  <entry name="product">STRIX Z270F GAMING</entry>
  <entry name="version">1.xx</entry>
  <entry name="serial">161290588201624</entry>
  <entry name="uuid">1300a340-695f-4ceb-8b2d-49dd0d1f2349</entry>
  <entry name="sku">SKU</entry>
  <entry name="family">To be filled by O.E.M.</entry>
  </system>
</sysinfo>
```
And in ```<os>``` add:
```
<smbios mode="sysinfo"/>
```
If you consider cpu pinning, this is my example for i7-7700k:
```
<vcpu placement="static">6</vcpu>
<iothreads>1</iothreads>
<cputune>
 <vcpupin vcpu="0" cpuset="1"/>
 <vcpupin vcpu="1" cpuset="5"/>
 <vcpupin vcpu="2" cpuset="2"/>
 <vcpupin vcpu="3" cpuset="6"/>
 <vcpupin vcpu="4" cpuset="3"/>
 <vcpupin vcpu="5" cpuset="7"/>
 <emulatorpin cpuset="0,4"/>
 <iothreadpin iothread="1" cpuset="0,4"/>
</cputune>
```
#### Don't forget to click apply multiple times so you don't forget anything!


## CREATING CPU ISOLATING SCRIPTS

#### This is optional, it may improve performance.


Create a 'isolstart.sh' script:
```
sudo nano /etc/libvirt/hooks/qemu.d/win10/prepare/begin/isolstart.sh
sudo chmod +x /etc/libvirt/hooks/qemu.d/win10/prepare/begin/isolstart.sh
```
Make it look like this, adapt to your liking:
```
systemctl set-property --runtime -- user.slice AllowedCPUs=0,4
systemctl set-property --runtime -- system.slice AllowedCPUs=0,4
systemctl set-property --runtime -- init.scope AllowedCPUs=0,4
```
Create 'isocpurevert.sh' script:
```
sudo nano /etc/libvirt/hooks/qemu.d/win10/release/end/iscpurevert.sh
sudo chmod +x /etc/libvirt/hooks/qemu.d/win10/release/end/iscpurevert.sh
```
Make it look like this, adapt it to your PC of course:
```
systemctl set-property --runtime -- user.slice AllowedCPUs=0-7
systemctl set-property --runtime -- system.slice AllowedCPUs=0-7
systemctl set-property --runtime -- init.scope AllowedCPUs=0-7
```


## SETTING UP OPENSSH ON A CLIENT

#### (Optional, not required).

#### Windows may come integrated with openssh available in cmd, if not, get it.

To make connecting to the hypervisor, host, we can make a 'config' file in ~/.ssh folder:
```
Host "name"
    HostName "ip_address"
    User "user"
```
This should enable connecting with just eg. ```ssh arch``` instead of ```ssh user@192.168.1.5```.

Thank you,

abdaiscool
