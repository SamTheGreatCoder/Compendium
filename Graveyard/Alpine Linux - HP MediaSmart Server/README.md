# Alpine Linux on HP MediaSmart Server

## Graveyard notice

After installing the additional packages and removing the extra firmware files, there were  only a couple MB remaining on the internal storage, meaning that upgrades to anything would require a rebuild of the entire configuration.

After consideration, this isn't worth the effort to maintain, even on an infrequent update cycle, and deleting firmware files as a means of fitting it on the storage, for a system that I consider critical, doesn't make sense. Something as simple as trying to update a single package would fill up the remainder of the storage and cause no free space errors.

I'm intending to switch to OpenWrt as the base distro. While ZFS is not present in the package repositories, `BTRFS` is available, and I plan to move the storage infrastructure in time. OpenWrt using less storage then Alpine also makes future maintainence much easier, and likely less problematic.

---

I acquired an old HP MediaSmart Server [EX470](https://en.wikipedia.org/wiki/HP_MediaSmart_Server#Models) and wanted to use it as my backup server (at the time of writing, my NAS is a "backup" server) to follow the [3-2-1 backup strategy](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/) more closely. However, any version of Windows Home Server is too old to keep running in my opinion, and considering the fact I use a large variety of platforms, locking my backup solution into something arguably propietary did not feel appropiate. I also didn't want the OS itself to be running off one of the hard drives, and wanted to be able to dedicate all four drive bays to purely storage, utilizing the internal "USB drive" to handle the operating system and any related programs.

With the internal storage being only ~256MB I need something lightweight that is still modern, updatable, extensible, and ideally familar and not overly niche. I knew I wanted Linux but finding a distro that would fit in the space would be difficult.

I ended up selecting Alpine Linux as it meets all of my desired criteria. Samba, NFS, and ZFS (yes really) are all supported by Alpine with really no effort. However, Alpine's default install size after firmware was well past the 256MB limit of the internal storage. By default, Alpine installs firmware for a plethora of devices, including ones irrelevent to the hardware of the MediaSmart Server. Including but not related to Intel CPU packages, ethernet for adapters other then SiS, audio support, and so on. Deleting irrelevant files from the `sys/kernel` folder frees up enough space to allow Alpine to fit and have storage for extra packages like ZFS, samba and NFS.

The specific flow to fit Alpine on the 256MB storage is to install Alpine on a larger flash drive first, (I used a 2GB drive), install all of the desired packages (samba, NFS, SSH, etc) and then install ZFS. I don't remember the reason but installing ZFS before clearing the extra firmware was needed to avoid broken file references.

These are the additional packages I installed:

```
amd-ucode
btop
cfdisk
dosfstools
e2fsprogs
e2fsprogs-extra
hdparm
linux-firmware-none
nfs-utils
ntfs-3g-progs (while I'm not using NTFS as the backing partition format, having it available as a "last-resort" means could be helpful)
rsync
smartmontools
zfs
zfs-lts
```

Once all the extra packages are installed, the following can be cleared from the `sys/kernel` directory:

```
/kernel/arch/x86
- crypto - (Anything aes,avx,intel,pclmul, etc. as it's not supported by the hardware)
- events/intel
- kvm - (anyting marked Intel)

/kernel/drivers/
- accessibility
- ata (not all need to remain here, unsure of which is utilized by the MediaSmart)
- atm
- auxdisplay
- bcma
- bluetooth
- char (virtio, ipmi, tpm, hw_random (anything not marked AMD)
- clk (not everything here may be required)
- cpufreq (speedstep-centrino for Intel)
- crypto (Intel, Virtio)
- dca
- dma (Qualcomm, virt-dma)
- edac (Non-AMD)
- extcon (USB-C)
- firewall
- firmware (Non EDD)
- gnss
- gpio (virtio)
- gpu
- hid (excluding amd-sfh-hid, i2c-hid, intel-ish-hid)
- iio
- infiniband
- input (Non-keyboard related, as there is no desktop environment or other input peripherals used)
- iommu
- isdn
- media
- memstick
- message
- mfd
- memstick
- mmc
- mtd
- mux
- net (appletalk, arcnet, bonding, can, dsa, ethernet (non-sim190), fddi, fjes, hippi, hyperv, ieee802154, plip, ppp, slip, team, thunderbolt, vmxnet3, wireguard, wireless, xen-netback, tap/tun/virtio)
- nfc
- nvdimm
- nvme
- parport
- pcmia
- platform (excluding /x86/amd)
- power
- powercap
- pwm
- regulator
- reset
- rpmsg
- scsi
- soc
- soundwire
- ssb
- staging
- thermal (intel specific)
- thunderbolt
- ufs
- usb (Type-C)
- virt
- virtio
- xen
/kernel/net
- 6lowpan
- 9p
- appletalk
- atm
- bluetooth
- can
- dsa
- ieee802154
- l2tp
- mac80211
- mac802154
- nfc
- openvswitch
- phonet
- qrtr
- rfkill
- rxrpc
- sunrpc
- vmw_vsock
- wireless
/kernel/sound
*
```

Following all of this, I shrink the size of the root filesystem to fit within the internal 256MB of storage, clone the partitions to the internal storage, and complete this.
