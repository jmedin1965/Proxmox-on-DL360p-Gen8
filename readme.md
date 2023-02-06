# Proxmox on DL360p Gen8


Created Friday 07 October 2022

The idea is to install proxmox on a DL360p Gen8. The problems are;

* The server is PCIe spot challenged. It only has 2 slots. a 16x and an 8x spot.
* It also has a built in slot with a 4x1G card.
* the built-in raid card Smart Array P420i that can't be flashed to IT mode
	* The raid array card can be switched to HBA mode but there are some warnings, so just don't know if this will work or not.
	* It does not work. Using RAID6 with LVM2 and this works much faster and does not need a CACHE module either.
* The caddy LED doesn't appear to work in HBA mode. It's a know limitation in HBA mode.
* The server does not support UEFI boot, only BIOS.
* Proxmox with Open Media Vault OVM installed on top works great.


Hardware required
-----------------

* 8 x 4TB seagate drives
	* Drives bought from Officeworks; 4TB expansion Partable Drive
	* $134 each
* 8 x disk caddies for the 8 drives above
	* <https://www.ebay.com.au/itm/152994582000>
	* 651687-001 HP Gen8 Gen9 2.5in SFF HDD SSD Drive Caddy Tray for DL180 DL380 G8 G9
	* $17.79 each
* 1 x 2 port 10G card to replace the 4x1G card
	* ~~https://www.ebay.com.au/itm/192513997944 - this one did not work~~
		* ~~$30.57~~
	* <https://www.ebay.com.au/itm/185607289358> - This one worked perfectly
		* 647581-B21 HP Ethernet 10Gb 2P 530FLR-SFP+ ADAPTER 649869-001 647579-001
		* $24.00
* ~~1 x Sandisk 32G USB 2.0 pen drive~~
	* ~~https://www.officeworks.com.au/shop/officeworks/p/sandisk-cruzer-blade-usb-flash-32gb-3-pack-sdcz5032p3~~
	* ~~$18.59~~
* ~~1 x SATA to USB cabe (only needed to make USB pen drive bootable, or to recover the system)~~
	* ~~https://www.ebay.com.au/itm/132712657279~~
	* ~~$7.46~~


Prep work
---------

### labels

stick labels on all the disk caddies so that when a disk fails, you know which one it is.

Found out the hard way that disk/by-id does not user the serial number but the WWN, which stands for World Wide Name. Make sure your lable has this number instead of serial number.

### Server fans too loud

If you have non HP hardware installed, then ilo software will run the fans loud. It's a known problem. There is a firmware hack that will let you tweak the temperature settings to slow down the fans to a normal speed. This YouTube explains it really well. It works but don't stuff the firmware upgrade up, as there is not coming back from a bricked ilo4.

#### YouTube tutorial and how to

shows how to set fan speed and power supply fans
REF: <https://www.reddit.com/r/homelab/comments/di3vrk/comment/firx6op/>

<https://www.youtube.com/watch?v=Keyz-9HNr7Q>

#### how to flash firmware
Tried to flash but have to switch ilo protection off to flash it.


1. [Download iLO4 2.50](https://support.hpe.com/hpsc/swd/public/detail?swItemId=MTX_42ef22e4dff6423e8dbe111904) CP027911.scexe We'll use this for flashing the hacked firmware
2. Download the custom 2.73 ROM We'll swap out the original firmware in the 2.50 iLO4.
3. Disable iLO security by way of the system maintenance switch on your motherboard, which is "switch 1 to on" REF: [System maintenance switch](https://techlibrary.hpe.com/docs/iss/DL380pGen8/setup_install/advanced/Content/137614.htm)
4. For Ubuntu, I had to do the following:
	a. sudo modprobe -r hpilo
5. Replace the 2.50 ROM with the [2.77 ROM](https://www.reddit.com/r/homelab/comments/sx3ldo/hp_ilo4_v277_unlocked_access_to_fan_controls/) and flash

sh ./CP027911.scexe --unpack=ilo_250
cd ilo_250
cp /path/to/ilo4_277.bin.fancommands ilo4_250.bin
sudo ./flash_ilo4 --direct
	
and now remote console works with HTML 5, which is great !!!!!!

#### Configuration  fan commands
We want to aim for 39%, which is normal for a standard server

ssh to ilo. If no display, reset ilo with "information>>diagnostics>>reset".

fan info g

look for fan number with a star. It is the one to the left.

fan info a

for me it was the set point (sp) for sensor 11. It was set to 46deg (4600) I slowly raised it to 54 (5400) and it settled on speed of 90

fan pid 11 sp 5400

fan pid 11 sp 5900
fan pid 42 lo 10000
fan pid 40 sp 4700
fan pid 46 sp 4400
fan pid 50 sp 3800
fan pid 31 sp 5300

The youtube said to disable it, but there was a comment about setting the target temperature.

And now I have 39% fan speed.

#### ilo connection timeout
ilo connection timeout can be set to infinite at Administration>> Access Settings>>Idle connection timeout (minutes)>>infinite
I disabled serial comunication timeout too, to see if this will stop loosing session display loss.

#### Scripts to update fans from ssh

REF: <https://github.com/That-Guy-Jack/HP-ILO-Fan-Control>

### The disk drives are slow in HBA mode

Need to test this. I have 5 Raidz2 disks. They work ok but have freeze hiccups. I will re create with 8 drives and test. If this fails, I will switch off HBA and install normal RAID6 and just put zfs on to of that and test this way, or just dump ZFS all together, but I'm used to using it now.

update: I just did a dd test on zfs and then one on a single drive formatted as fat32. Just a single drive is way faster that raidz2 on 5 drives. I'm deffinitely going to test with just raid6. Also, it looks like ovm does work installed on top of proxmox and the zfs module works too. Just not with zfs on the rootfs. It causes problems. So will re-install proxmox using ext4 on the raid6 array. This will make it bootable again and the disk LED's will work too. I will use a small partition and then install zfs on the rest of the disk. Probably 256G as root and 256 of swap and the rest zfs.

update: removed HBA mode, created RAID6 with the 8 drives. Now we do not need a USB drive to boot and LVM2 is much faster that ZFS by a factor of 4. LVM2 is the way to go.

### Ethernet 10Gb 2-port 530FLR-SFP+ Adapter
<https://pci-ids.ucw.cz/read/PC/14e4/168e>

 Vendor 14e4 -> Device 14e4:168e

1014 0492	PCIe2 2-port 10 GbE BaseT RJ45 Adapter (FC EN0W; CCIN 2CC4)	
103c 1798	Flex-10 10Gb 2-port 530FLB Adapter [Meru]	
103c 17a5	Flex-10 10Gb 2-port 530M Adapter	
103c 18d3	Ethernet 10Gb 2-port 530T Adapter	
103c 1930	FlexFabric 10Gb 2-port 534FLR-SFP+ Adapter	
103c 1931	StoreFabric CN1100R Dual Port Converged Network Adapter	
103c 1932	FlexFabric 10Gb 2-port 534FLB Adapter	
103c 1933	FlexFabric 10Gb 2-port 534M Adapter	
103c 193a	FlexFabric 10Gb 2-port 533FLR-T Adapter	
103c 3382	Ethernet 10Gb 2-port 530FLR-SFP+ Adapter	
103c 339d	Ethernet 10Gb 2-port 530SFP+ Adapter	
193d 1003	530F-B	
193d 1006	530F-L	
193d 100f	NIC-ETH522i-Mb-2x10G

No luck with this card but I think it's just a faulty card.

Switch raid controller to hba
-----------------------------

### The boot disk to do the conversion
I used ubuntu-22.04.1-desktop-amd64.iso on a ventoy boot USB. I have the iso boot a persistent.

REF <https://www.ventoy.net/en/plugin_persistence.html#:~:text=Many distros (like Ubuntu%2FMX,time you boot to it>.

There is a trick to creating the persistent dat/bin file though so read the instructions from the

 
URL above "Backend Image File".

edit or create \ventoy\ventoy.json

"persistence": [
{
"image": "/ISO/ubuntu/ubuntu-20.04-desktop-amd64.iso",
"backend": [
"/persistence/ubuntu_20.04_1.dat"
],
"autosel": 1,
"timeout": 20
},
{
"image": "/ISO/ubuntu/ubuntu-22.04.1-desktop-amd64.iso",
"backend": [
"/persistence/ubuntu_22.04_1.dat"
],
"autosel": 1
}
]
}

### The conversion
REF: <https://forums.unraid.net/topic/82007-solved-unraid-with-hp-p420i-raid-card-in-hp-proliant-dl380p-g8/>

* make bootable persistent ubuntu install, see above using ventoy
* boot into ubuntu
* download hpssacli-2.10-14.0_amd64.deb from <https://downloads.linux.hpe.com/SDR/repo/mcp/pool/non-free/>
* dpkg --install hpssacli-2.10-14.0_amd64.deb
* cd [/opt/hp/hpssacli/bld](file:///opt/hp/hpssacli/bld)
* [./hpssacli](./readme_files/Proxmox_on_DL360p_Gen8/hpssacli)
* ![](./readme_files/Proxmox_on_DL360p_Gen8/pasted_image001.png)


Installation
------------
Below is how to install proxmox with controller in HBA mode. This requires a USB drive to keep the boot partition. Using the disks in RAID 6 and installing LVM2 menas you don't need a USB stick. The disk array is now bootable and much fastre.

Since only USB is bootable and not the HBA disk drives. I am going to create a RAIDZ2 array with the first 6 disks. I plan to leave the other 2 for TrueNas. I will limit the ROOT rpool to 128G, which will give rpool of 512G. I will also add another partition to each of the 8 drives for swap.

So, about swap. I have read a lot about not having swap and that RAM is so cheap, why should you slow down your server by having slow swap;

* server RAM is not cheap
* setting swappiness in thge kernet to 0 does not disable swap, like it's mentioned on the internets. These people have no clue.
* In the past, as a rule of thump, for a running app, 80% of the time is spent in 20% of the code. Now, this will not be the case now because app now use a lot more data. But even if this is 80% of the time is spen in 50% of the code then, you are wasting that 50% of ram used. The Linux kernel is written very efficiently. It uses swap efficiently. During startup and application initialises a lot and then never looks at this again. Why have this code sitting in valuable RAM. Why not lt the efficient kernel just swap it to disk where it will never be called for again.


### Install proxmox

* again using ventoy boot the proxmox iso which is proxmox-ve_7.2-1.iso for me now.
* You may need to tweak the BIOS to be able to boot from USB. I set external USA has priority so that ventoy USB will boot if installed. If not the 32G internal thumb drive will boot. Yes, the server has one internal USB. All the USB ports are version 2.0.
* unselect the internal USB and external USB and choose the first 6 4TB scsi drives.
* For the format choose ZFS RAIDZ2. This lets you loose 2 disks and still have a working system
* click on advanced and set the size to 128G
* now install and set the system name and email address and IP address.
* the system won't boot, just power it off now
	* pop out one of the disk caddies
	* connect the sata to USB to it.
	* plug it in to where the vertoy boot disk was
	* boot the system. It should boot to the USB HDD as normal now.
	* Follow the instructions for replacing a failed boot disk, but we are not replacing one. We are just adding one
		* REF:  <https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#_zfs_administration>
			* Changing a failed bootable device
			* "proxmox-boot-tool status" should show all your boot disks (6 in total)
			* "cat [/proc/partitions"](file:///proc/partitions%22) this will list all partitions so you can work out which is the USB pen drive. You should only have one installed of size 32G. All others will be 4TB.
			* "sgdisk <healthy bootable device> -R <new device>"
				* example, but don't use this, work out what the device names should be for you "sgdisk /dev/sda -R [/dev/sdg"](file:///dev/sdg%22)
			* "sgdisk -G <new device>" don't forget this step. I did once and had bad things happen since two of my boot partitions had the same UUID.
			* so, you will get complaints about partition3 since it just doesn't fit. Ignore this since we will delete it
			* "fdisk [/dev/sdg"](file:///dev/sdg%22) please use your device and don't jusy copyu the example here
				* d 3
				* M
				* p
					* make sure the partition has a * under boot. If not press "a", which toggles the bootable flag. I had to do this to make the USB boot. Funny, it's not just for proxmox. Other people complain about truenas on a USB peodrive install having the same problem. But if the USB device is a HDD, it boots without it. Ok, and it only affects BIOS boot, not UEFI.
				* w
				* q
			* "proxmox-boot-tool format <new disk's ESP> --force" 
				* i had to use --force to get it to format.
				* this partition is partition2, so as an example "proxmox-boot-tool format /dev/sdg2 --force
			* proxmox-boot-tool init <new disk's ESP>
			* update-grub
				* I think it runs this anyway automagically but it can't hurs.It takes a while because it does it on all 7 drives. This is run every time grub is updated, so when you do alt update/upgrade.
				* oh, ok, it just runs "proxmox-boot-tool refresh", so just run that then.
		* so now the internal USB should be bootable so if you reboot it should boot from the internal USB.
		* "poweroff" the server
		* put back the 4TB caddy
		* power the server back on
		* if it boots then great. things to check;
			* cat [/etc/kernel/proxmox-boot-uuids](file:///etc/kernel/proxmox-boot-uuids)
				* this will list the UUID of all your boot disks. You should have 7. 6 disks and 1 USB pen drive
			* ls -l [/dev/disk/by-uuid](file:///dev/disk/by-uuid)
				* this will list the UUID if all disks. Plese compare. If they don't match up. You can add the missing one. [/etc/kernel/proxmox-boot-uuids](file:///etc/kernel/proxmox-boot-uuids) is just a text file.
				* proxmox-boot-tool has lots of options; refresh, kernel list, status
		* tidy up steps (best you learn how to manage zfs now so you don't break a production system). Oh and best to turn on zfs autocomplete for bash
			* run "zpool status" and make all the raidz2 disks start with scsi-. the disk I user to boot from swapped to using sde3 which is bad. You get a warning when the server boots. It is easily repairable.
				* ls -l [/dev/disk/by-id/scsi-*](file:///dev/disk/by-id/scsi-*)
				* find the one that matched the device and get the scsi- name instead.



Post install stuff
------------------

### DMAR: DRHD: handling fault status reg 2

Error on console log after boot. 

REF: <https://bugzilla.kernel.org/show_bug.cgi?id=202723>

Fix by doing the following, add intel_iommu=igfx_off to kernel command line

### NVIDIA GPU Driver

on a root terminal;

apt install nvidia-driver
ls [/dev/nvidia](file:///dev/nvidia)


