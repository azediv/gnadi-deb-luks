# Howto Luks Debian and Gandi

This Howto describe a process to create a raw vm of Debian with an encrypted root partition on Gandi servers.
Step 1, 2 and 5 are common to other distribution.

1. creation of a vm with all tools to start minimal installation of "any" distribution on Gandi IaaS VPS/servers
2. creation of target disk that will be use as target disk and is needed for "any" distribution.
3. specific to Debian installation with LVM encrypted in kvm vm with raw format
4. specific to encrypted volume and Grub bootloader configuration
5. configure Gandi VPS with new disk and is for "any" distribution

## 1 : Gandi vm creation, tools install and raw vm creation.

We will proceed to a Debian vm creation and tools required to create a qemu/kvm raw vm inside of this Gandi vm. (sexy isn't it ?)
Then we will download desired boot medium for an install. A netinstall iso of latest debian version in this example.

### 1.1 : Creation of temp vm :

	local $ gandi vm create --hostname vdl --datacenter LU --ip-version 4 --login user --password --memory 8192 --cores 4 --image 'Debian 8 64 bits (HVM)' --size 10G
	local $ gandi vm ssh vdl

### 1.2 : Installation of requirements on temp vm :

	vm # apt-get update && apt-get upgrade -y && apt-get install kvm qemu-kvm libvirt-bin virtinst bridge-utils virt-manager wget cryptsetup -y

### 1.3 : Creation of working directory

	vm # mkdir /home/vm
	vm # cd /home/vm

### 1.4 : Download of debian netinstall iso

	vm # wget http://cdimage.debian.org/debian-cd/8.2.0/amd64/iso-cd/debian-8.2.0-amd64-netinst.iso

### 1.5 : Creation of raw image for installation of raw vm

	vm # qemu-img create -f raw deb.raw 3G

### 1.6 : Configuration of network on temp vm for raw vm

Edit gandi config file :

	vm # nano/vi(m) /etc/default/gandi

and change : 

	"CONFIG_NETWORK=1"

into : 

	"CONFIG_NETWORK=0"

then configure bridge network : 

	vm # virsh iface-bridge eth0 br0

Here you'll be disconnected so :

	local $ gandi vm reboot vdl
	local $ gandi vm ssh vdl

### 1.7 : Creation of raw vm with iso as boot disk 

	vm # cd /home/vm
	vm # virt-install --connect qemu:///system -n vm -r 2048 --vcpus=1 --disk  path=/home/vm/deb.raw -c /home/vm/debian-8.2.0-amd64-netinst.iso --vnc  --noautoconsole --os-type linux --network=bridge:br0 --hvm
	
### 1.8 : Connexion to raw vm, 2 solutions :

#### 1.8.1 : Install virt-manager on local computer

	local $ sudo apt-get install virt-manager
	local $ virt-manager

Choose in Menu : File => Add a connexion

Hypervisor : QEMU/KVM
tick "Connexion to a remote host"
Method :  SSH
Username : root 
Hostname : IP of temp VM
Select Automatic connexion and click "Connect"	

In virt-manager, right-click on new connexion and click "Connect"
You will then be able to select raw vm and click "Open" in menu.

#### 1.8.2 : Creation of a tunnel to initiate vnc connection from local computer 
	local $ ssh user@VDL_IP -L 5900:127.0.0.1:5900
	local $ vncviewer 127.0.0.1

select "Install" in viewer and you'll be disconnected so connect again

	local $ vncviewer 127.0.0.1

### 1.9 : Manual config of network during installation with vm public network settings 

from temp vm :

	IP : cat /gandi/config| grep -Po '(?<="pna_address": ")[^"]*' |head -1
	GATEWAY : cat /gandi/config | grep -Po '(?<="pbn_gateway": ")[^"]*' | head -1
	DNS : cat /etc/resolv.conf | awk '{ print $2}' | grep -E -o "([0-9]{1,3}[\.]){3}[0-9]{1,3}" | head -1
	HOSTNAME : cat /gandi/config| grep -Po '(?<="vm_hostname": ")[^"]*'

## 2 : Target disk creation

This section is for any distribution

### Creation of target disk

	local : $ gandi disk create --size 4G --name vdluks --vm vdl
	vm : # umount /dev/sdc

## 3 : Debian install with LVM encrypted 

This section is specific to Debian install and should be adapted for another distribution

### 3.1 : Partitionning 

select assisted all in one disk with encrypted LVM

all in one partition (recommend for beginner)

### 3.2 : Networking

**/!\ important note /!\**

after apt mirror selection, when it fail to find network mirrors :
choose "Go Back" and continue without network mirrors with "Yes"

	irl : grab a coffee

### 3.3 : Bootloader

at the end of installation, choose to install Grub "Yes" and on /dev/sda

When install is over, kvm vm will stop due to default settings "no autostart"

## 4 : Configuraiton of Grub bootloader

Copy to disk and mount with cryptsetup to reinstall Grub

### 4.1 : Copy image to disk and mount partitions to chroot : 

Copy of raw image with dd to target disk :

	vm # dd if=deb.raw of=/dev/sdc

Open with cryptsetup root partition and name it vm :

	vm # cryptsetup luksOpen /dev/sdc5 vm

Create mount directory :

	vm # mkdir /srv/vm

Scan all pĥysical volume :

	vm # vgscan
	vm # vgchange -ay

Scan all logical volume : 

	vm # lvscan

Note logical volume path to mount it.

Mount {root,boot,dev,proc,sys} partitions :

	vm # mount /dev/deb-vdl/root /srv/vm
	vm # mount /dev/sdc1 /srv/vm/boot/
	vm # mount -o bind /dev/ /srv/vm/dev/
	vm # mount -o bind /proc/ /srv/vm/proc/
	vm # mount -o bind /sys /srv/vm/sys/

Chroot inside new debian vm :

	vm # chroot /srv/vm

### 4.2 : Edit grub : 

	chroot # vi(m)/nano /etc/default/grub 

and change :

	# GRUB_CMDLINE_LINUX=""

to :

	# GRUB_CMDLINE_LINUX="console=ttyS0"

### 4. 3 : Reinstall Grub :

	chroot # grub-install /dev/sdc
	chroot # update-grub
	chroot # exit

	vm # exit

## 5 : The end

### 5.1 : Detach, update, attach and start vm : 

	local $ gandi vm stop vdl
	local $ gandi disk detach sys_vdl -f
	local $ gandi disk detach vdluks -f
	local $ gandi disk update vdluks --kernel raw
	local $ gandi disk attach vdluks -p 0 vdl -f
	local $ gandi vm start vdl

### 5.2 : Decrypt Partition with console :

	local $ gandi vm console vdl

	Asking for console, please wait
	Connected

	Grabbing terminal
	Ok
	...
	Please unlock disk sda5_crypt:

Unlock !!!

Login as root for next step.

vm has network, please ping us !

	raw_vm # ping gandi.net 

## Optimization : 

### Edit sources :

	raw_vm # # vi(m)/nano /etc/apt/sources.list

And add

	deb ftp://ftp.fr.debian.org/debian/ jessie main contrib non-free
	deb ftp://ftp.fr.debian.org/debian/ jessie-updates main contrib non-free

### Update 

	raw_vm # apt-get update

### Install gandi-hosting-vm2 :

	raw_vm # wget http://mirrors.gandi.net/gandi/debian/pool/gandi-hosting-vm2_2.6_all.deb
	raw_vm # dpkg -i gandi-hosting-vm2_2.6_all.deb
	raw_vm # apt-get install -f

### Install OpenSSH server : 

	raw_vm # apt-get install openssh-server
	local $ ssh-keygen -f "/home/$USER/.ssh/known_hosts" -R VM_IP
	local $ gandi vm ssh --login user vdl

Ready !
