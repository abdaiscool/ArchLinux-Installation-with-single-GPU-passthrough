# INTRODUCTION

Hello,

I got motivated by SomeOrdinaryGamers, Mutahar, to attempt to make a simple to use VM, hidden from the guest, and quick enough to play games, so I've made this guide with my steps to make it work, this is primarly for me so I don't forget how to do this again.

I may not be doing this in the best method possible, but this is how it worked for me, so you are free to make adaptions and change to your liking.

I also just made this for myself incase I ever want to go back to this project. I've finished it but I don't feel at home yet, so I'm going back to Windows for now.

Thanks and I hope this helped anyone who needed help.


# SOURCES


[The Passthrough post](https://passthroughpo.st)

[Single-GPU-Passthrough](https://github.com/joeknock90/Single-GPU-Passthrough) by joeknock90 on Github

[PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) by wiki.archlinux.org

[I Made The Greatest Windows 11 Virtual Machine...](https://www.youtube.com/watch?v=WYrTajuYhCk)  by SomeOrdinaryGamers on Youtube

[I Almost Lost My Virtual Machines...](https://www.youtube.com/watch?v=BUSrdUoedTo) by SomeOrdinaryGamers on Youtube

[Indian Man Beats VALORANT's Shady Anticheat...](https://www.youtube.com/watch?v=L1JCCdo1bG4) by SomeOrdinaryGamers on Youtube



## INSTALLATION OF ARCH LINUX


entering the fdisk utility:
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

#### You would want to make a first partition with the last sector: +500M

#### Select 1: 'EFI partition'

#### You would want to make a second partition and leave the type as 'Linux filesystem'


format the first partition as FAT32:
```
mkfs.fat -F32 /dev/nvme0n1p1 
```
format the second partition as ext4:
```
mkfs.ext4 /dev/nvme0n1p2
```
mount the second partition:
```
mount /dev/nvme0n1p2 /mnt
```
create a home folder:
```
mkdir /mnt/home
```
creating a fstab file:
```
mkdir /mnt/etc
genfstab -U -p /mnt >> /mnt/etc/fstab
```
confirm everything is good:
```
cat /mnt/etc/fstab
```
install a package group / base packages:
```
pacstrap -i /mnt base
```
to configure the installation, to use the installation as chroot:
```
arch-chroot /mnt
```
install linux, for long-term support: ```linux-lts linux-lts-headers```
```
pacman -S linux linux-headers  
```
install packages like nano, networkmanager... for wifi consider: ```dialog wpa_supplicant wireless_tools netctl```
```
sudo pacman -S base-devel openssh nano networkmanager 
```
enable NetworkManager & openssh to start when the computer starts:
```
systemctl enable NetworkManager
systemctl enable sshd
```
generate mkinitcpio files:
```
mkinitcpio -p linux (linux-lts)
```
time to select locales, uncomment the locales you need. I picked these: ```hr_HR.UTF-8 UTF-8 & en_us.UTF-8 UTF-8```
```
nano /etc/locale.gen
```
generate locales:
```
locale-gen
```
setting the root password:
```
passwd
```
creating the user:
```
useradd -m -g users -G wheel "username"
```
setting the password for the user:
```
passwd "username"
```
enabling sudo for users, uncomment ```%wheel ALL=(ALL) ALL```
```
EDITOR=nano visudo
```
getting grub for UEFI:
```
pacman -S grub efibootmgr dosfstools os-prober mtools
```
create EFI folder:
```
mkdir /boot/EFI
```
mount EFI partition:
```
mount /dev/nvme0n1p1 /boot/EFI
```
installing grub:
```
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
```
copying locales:
```
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```
edit default/grub:
under ```GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"``` add ```intel_iommu=on```

so it looks like this: ```GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 intel_iommu=on quiet"```

you also need to enable VD-t on intel systems in bios.
```
nano /etc/default/grub
```
generate grub file:
```
grub-mkconfig -o /boot/grub/grub.cfg
```
unmount drives and reboot, you should be able to boot without the installation media:
```
umount -a
reboot
```
set the timezone, I set in my case ```Europe/Zagreb```:
```
timedatectl set-timezone Europe/Zagreb
```
enabling the synchronization of clock:
```
systemctl enable systemd-timesyncd
```
setting the prefered keyboard layout for console and x11 (```croat``` & ```hr``` are for croation, use as wanted):
```
nano /etc/vconsole.conf
```
in this newly created vconsole.conf add ```KEYMAP=croat```.
```
localectl set-keymap croat
localectl set-x11-keymap hr
```
setting the hostname:
```
hostnamectl set-hostname "name"
nano /etc/hosts
```
add these lines at the end of the file:
```
127.0.0.1 localhost
::1 localhost
127.0.1.1 "name"
```
installing the microcode, xorg, nvidia (nvidia-lts for linux-lts):
```
pacman -S intel-ucode xorg-server nvidia nvidia-utils
```
installing GNOME:
```
pacman -S gnome gnome-tweaks
```
enabling display manager:
```
systemctl enable gdm
```
installing yay, AUR helper:
```
sudo pacman -S git
cd /opt
sudo git clone https://aur.archlinux.org/yay-git.git
sudo chown -R "username":users ./yay-git
cd yay-git
makepkg -si
```


## SOFTWARE FOR KVM/QEMU VM


confirm IOMMU is enabled:
```
dmesg | grep -i -e DMAR -e IOMMU
```
indentify your gpu, make sure it is in its separe iommu group, as well as anything else you want to pass:
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
#### copy all your devices in your GPU to a seperate file to use for later

#### if there is anything else in your IOMMU group, and it isn't PCI BRIDGE, you may want to consider ACS patching.

install all required software for qemu:
```
sudo pacman -S qemu libvirt edk2-ovmf virt-manager iptables-nft dnsmasq 
```
enable and start the services:
```
sudo systemctl enable libvirtd.service virtlogd.socket
sudo systemctl start libvirtd.service virtlogd.socket
```
activate and start the default libvirt network:
```
virsh net-autostart default
virsh net-start default
```


## DUMPING GPU VBIOS + PATCH


get nvflash from yay:
```
yay -S nvflash 
```
get bless (hex editor):
```
sudo pacman -S bless
```
dump your vbios (you may want to do this with open-source drivers):
```
sudo nvflash --save gpu.rom
```
using bless, load 'gpu.rom' and search for 'VIDEO' as text.
a little earlier then 'VIDEO' you will find a 'U'. delete everything that comes before it.

save as 'patch.rom'


## CREATING A VM USING VIRT-MANAGER


1. download windows iso from microsoft
2. open virt-manager
3. create a new vm
4. select local instal media
5. if it asks for permissions, click yes
6. give it ram
7. give it cpu
8. you may want to use a virtual disk, or passthrough a disk
9. select customize before install
10. change chipset to 'Q35'
11. change firmware to 'UEFI x86_64...OVMF_CODE.fd'
12. setup topology
13. enable 'SATA CDROM1' in boot options
14. press next to install windows...
15. when you finish, shut it down


## SETTING UP A SCRIPT TO HIJACK AND RELEASE THE GPU


get wget:
```
sudo pacman -S wget
```
make a libvirt/hooks folder:
```
sudo mkdir -p /etc/libvirt/hooks
```
install hook manager and make it executable:
```
sudo wget 'https://raw.githubusercontent.com/PassthroughPOST/VFIO-Tools/master/libvirt_hooks/qemu' \
     -O /etc/libvirt/hooks/qemu
sudo chmod +x /etc/libvirt/hooks/qemu
```
make a kvm.conf file:
```
nano /etc/libvirt/hooks/kvm.conf
```
add this into the 'kvm.conf' (you copied the numbers when you identified the gpu):
```
VIRSH_GPU_VIDEO=pci_0000_01_00_0
VIRSH_GPU_AUDIO=pci_0000_01_00_1
```
get tree:
```
sudo pacman -S tree
```
#### you need to create scripts and folder to look like this:
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
first script to hijack the gpu:
```              
sudo mkdir /etc/libvirt/hooks/qemu.d/win10/prepare/begin
sudo nano /etc/libvirt/hooks/qemu.d/win10/prepare/begin/start.sh
```
the start.sh script (the script is personalized towards me):
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
make the script executable:
```
sudo chmod +x /etc/libvirt/hooks/qemu.d/win10/prepare/begin/start.sh
```
second script to release the gpu:
```
sudo mkdir /etc/libvirt/hooks/qemu.d/win10/release/end
sudo nano /etc/libvirt/hooks/qemu.d/win10/release/end/revert.sh
```
the revert.sh script (the script is personalized towards me):
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
make the script executable:
```
sudo chmod +x /etc/libvirt/hooks/qemu.d/win10/release/end/revert.sh
```


## PASSING THE GPU TO THE VM


1. go into virt-manager and add your gpus under '+Add Hardware' and 'PCI Host Device'
2. edit the added PCI device within virt-manager as XML
3. under ```</source>``` and above ```<address type='pci'...>```
4. add ```<rom file='/home/"user"/patch.rom'>``` (you should change this path the the patch.rom path you made earlier)
5. you should add this line to all your GPU devices

#### before booting the VM add following lines to your VM's xml (win10.xml in my case):

inside ```<hyperv>```:
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
#### after installing windows, attempt to enable Hyper-V in Windows features, it might help with anticheats.

under ```</hyperv>``` add:
```
<kvm>
 <hidden state="on"/>
</kvm>
```
change the ```<cpu>``` section:
```
<cpu mode="host-passthrough" check="none" migratable="on">
 <topology sockets="1" dies="1" cores="3" threads="2"/> (suit your topology)
 <feature policy="disable" name="hypervisor"/>
 <feature policy="require" name="topoext"/>
</cpu>
```
above ```<os>``` add:
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
and in ```<os>``` add:
```
<smbios mode="sysinfo"/>
```
if you consider cpu pinning, this is my example for i7-7700k:
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
#### don't forget to click apply multiple times so you don't forget anything


## CREATING CPU ISOLATING SCRIPTS


create a isolstart.sh script
```
sudo nano /etc/libvirt/hooks/qemu.d/win10/prepare/begin/isolstart.sh
sudo chmod +x /etc/libvirt/hooks/qemu.d/win10/prepare/begin/isolstart.sh
```
make it look like this, adapt to your liking:
```
systemctl set-property --runtime -- user.slice AllowedCPUs=0,4
systemctl set-property --runtime -- system.slice AllowedCPUs=0,4
systemctl set-property --runtime -- init.scope AllowedCPUs=0,4
```
create isocpurevert.sh script
```
sudo nano /etc/libvirt/hooks/qemu.d/win10/release/end/iscpurevert.sh
sudo chmod +x /etc/libvirt/hooks/qemu.d/win10/release/end/iscpurevert.sh
```
make it look like this, adapt it to your PC of course:
```
systemctl set-property --runtime -- user.slice AllowedCPUs=0-7
systemctl set-property --runtime -- system.slice AllowedCPUs=0-7
systemctl set-property --runtime -- init.scope AllowedCPUs=0-7
```


## SETTING UP OPENSSH ON A CLIENT


#### Windows may come integrated with openssh available in cmd, if not, get it.

to make connecting to the hypervisor, host, we can make a 'config' file in ~/.ssh folder:
```
Host "name"
    HostName "ip_address"
    User "user"
```
this should enable connecting with just eg. ```ssh arch``` instead of ```ssh user@192.168.1.5```.

Thank you,

abdaiscool

