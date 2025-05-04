# Debian 12 - Unlock LUKS volume with YubiKeys

I wanted to use my YubiKeys to unlock my LUKS encrypted drives on Debian 12.  To do this, I needed to switch the initramfs generator to dracut.  Below are the steps I followed:

---

### Switch from initramfs-tools to dracut to manage initrd.

```
apt install dracut -y
apt purge cryptsetup-initramfs && apt autoremove --purge -y
echo hostonly=yes > /etc/dracut.conf.d/10-hostonly.conf
dracut -f
```

### Test that dracut is working by rebooting.

```
sudo reboot
```

### Once successfully booted using dracut, verify the partition that is LUKS encrypted using 'lsblk'.

```
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1             259:0    0 931.5G  0 disk  
├─nvme0n1p1         259:1    0   976M  0 part  /boot/efi
├─nvme0n1p2         259:2    0   977M  0 part  /boot
└─nvme0n1p3         259:3    0 929.6G  0 part  
  └─nvme0n1p3_crypt 254:0    0 929.6G  0 crypt 
    ├─deb12-root    254:1    0  97.7G  0 lvm   /
    ├─deb12-swap    254:2    0   1.9G  0 lvm   [SWAP]
    └─deb12-data    254:3    0   830G  0 lvm   /mnt/deb12
```

You can see the partition that that has the type 'crypt' is 'nvme0n1p3'.  Take note as you will use this to enroll your YubiKeys in the next steps.

### Install fido2-tools.

```
apt install fido2-tools -y
```

### Verify your YubiKey is seen and can be used to unlock your LUKS encrypted drive.

```
systemd-cryptenroll --fido2-device=list
```

### If your YubiKey is shown in the output, you can proceed with enrolling it to the disk partition from above.  You may need to run this with sudo.

```
systemd-cryptenroll /dev/nvme0n1p3 --fido2-device=auto --fido2-with-client-pin=yes
```

You will be presented with a series of prompts:
- Enter the current passphrase for the disk /dev/nvme0n1p3:
- Please enter the YubiKey PIN:
- Please confirm presence by touching YubiKey:

It should then let you know if your FIDO2 YubiKey was successfully enrolled.

*** You should be able to enroll multiple keys

### Edit the /etc/crypttab file as shown below:

```
nano /etc/crypttab

Change the line:

nvme0n1p3_crypt UUID=124f9dkb-67f3-29d8-aa55-93158d500964 none luks,discard

to:

nvme0n1p3_crypt UUID=124f9dkb-67f3-29d8-aa55-93158d500964 none luks,discard,fido2-device=auto
```

### Update initrd to include FIDO2 tools and updated /etc/crypttab

```
dracut -f
```

### Make sure your YubiKey is plugged in and test that everything is working by rebooting

```
sudo reboot
```

### You will either be presented with a screen asking you to enter your YubiKey PIN and then you will need to touch the YubiKey to confirm presence, or your bootup will appear to have halted.  You will need to make sure NumLock is enabled and start entering your YubiKey PIN, and then touch the YubiKey to confirm presence.  The bootup process will then proceed.
