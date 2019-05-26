# How to recover a failed Memup NS4600p - PROMISE 4-bay NAS ?

Today we will investigate and fix a failed Memup nas, 

    U-Boot 1.3.4 (NS4600p - 015 - 800MHz) (Jan 27 2011 - 10:21:32)
    
    CPU:   AMCC PowerPC 431EXr at 800 MHz (PLB=200, OPB=100, EBC=100 MHz)
           Security/Kasumi support
           Bootstrap Option F - Boot ROM Location NAND (8 bits), booting from NAND
           Internal PCI arbiter disabled
           32 kB I-Cache 32 kB D-Cache
    Board: NS4600p - PROMISE 4-bay NAS Target Board, 1*PCIe/1*SATA
    I2C:   ready
    DRAM:  256 MB
    Enclosure: Load fan configurations from VPD
    NAND:  128 MiB
    eth0 MAC = 00:01:55:30:6d:7e
    eth1 MAC = 00:00:00:00:00:00
    PCI:   Bus Dev VenId DevId Class Int
    PCIE1: successfully set as root-complex
            01  00  105a  3f20  0104  00
    SCSI:  Net:   ppc_4xx_eth0
    
    

with **PPC460E RAID-5 /  PDC42819 Controller** proprietary raid card.

    Loading modules:
    t3sas - t3sas 0000:81:00.0: enabling device (0006 -> 0007)
    t3sas 0000:81:00.0: Found PDC42819 Controller 105a:3f20 with IRQ: 18
    t3sas 0000:81:00.0: Driver version of PDC42819 : 1.3.0.14-NAS-15


Our motherboard is a rebranded promise ns4300 labeled ns4600p with a powerpc build, now memup is gone and we can't find any firmware to recover the nas.

This pcie is bundled with a proprietary raid card controller PDC42819 and you can find a copy of the driver but you have to build it [driver](url)

Fortunatly, we can easily build an old debian system with ppc support to try nand recovery :)


# Steps

 1. Connect Serial a TTL / USB ([Amazon]( https://www.amazon.fr/DSD-TECH-Convertisseur-broches-Compatible/dp/B072K3Z3TL/ref=sr_1_2_sspa?__mk_fr_FR=%C3%85M%C3%85%C5%BD%C3%95%C3%91&keywords=Cp2102&qid=1558850926&s=computers&sr=1-2-spons&psc=1))
 2. Enter in recovery mode with Uboot
 3. Recover custom configuration
 4. Create your own NFS server
 5. Create your own Ramdisk from fresh **chroot**
 6. Dump faulty nand and search for drivers
 7. Boot recovered Ramdisk !
 8. Creditz
 
## Connect Serial

Here is the pinout of this motherboard **GP0930-03 rev.A3**
![Serial pinout](https://i.postimg.cc/mkhw7J7L/pinout.jpg)

Configure your terminal to read at speed 115200 baud

![putty](https://i.postimg.cc/j2dscQjd/2019-05-26-08-34-08-COM5-Pu-TTY.png)

Now we can conclued that the nand working but the ramdisk don't pass the CRC (Maybe nand corruption or faulty update …)

    NAND read: device 0 offset 0xc80000, size 0x300000
     3145728 bytes read: OK
    
    NAND read: device 0 offset 0xf80000, size 0x800000
     8388608 bytes read: OK
    
    NAND read: device 0 offset 0x100000, size 0x3000
     12288 bytes read: OK
    ## Booting kernel from Legacy Image at 01200000 ...
       Image Name:   Linux-2.6.32.14
       Created:      2011-01-20   4:39:31 UTC
       Image Type:   PowerPC Linux Kernel Image (gzip compressed)
       Data Size:    2339544 Bytes =  2.2 MB
       Load Address: 00000000
       Entry Point:  00000000
       Verifying Checksum ... OK
    ## Loading init Ramdisk from Legacy Image at 01b00000 ...
       Image Name:   02.01.4300.19
       Created:      2011-04-22   2:05:52 UTC
       Image Type:   PowerPC Linux RAMDisk Image (gzip compressed)
       Data Size:    3714991 Bytes =  3.5 MB
       Load Address: 00000000
       Entry Point:  00000000
       Verifying Checksum ... Bad Data CRC
    Ramdisk image is corrupt or invalid
  
    
   ## Recover custom configuration

Now we have an access to uboot, lets configure some stuff

    setenv bootdelay 10 # startup delay to press Ctrl+C
    setenv ipaddr 192.168.1.31 # ip of nas 
    setenv gatewayip 192.168.1.254 # ip of my box
    saveenv # save change
    
Check partitions with uboot :

    => mtdparts
    
    device nand0 <nand0>, # parts = 10
     #: name                size            offset          mask_flags
     0: u-boot              0x00100000      0x00000000      0
     1: dtb                 0x00080000      0x00100000      0
     2: safe-k              0x00300000      0x00180000      0
     3: safe-r              0x00800000      0x00480000      0
     4: kernel              0x00300000      0x00c80000      0
     5: rootfs              0x00800000      0x00f80000      0
     6: usr                 0x01000000      0x01780000      0
     7: data                0x00200000      0x02780000      0
     8: oem                 0x00100000      0x02980000      0
     9: app                 0x05580000      0x02a80000      0
    
    active partition: nand0,0 - (u-boot) 0x00100000 @ 0x00000000
    
    defaults:
    mtdids  : nand0=nand0
    mtdparts: mtdparts=nand0:1024K(u-boot),512K(dtb),3072K(safe-k),8192K(safe-r),3072K(kernel),8192K(rootfs),16384K(usr),2048K(data),1024K(oem),-(app)
    
 Partitions are here, let's try to recovery data partitions …
Boot to recovery mode :

    # enter in shell mode at boot no need root password
    setenv addtty ${addtty} ${mtdparts} init=/bin/sh
    run safeboot
    




## Recover custom configuration

Now we have a minimal busybox environment, we will mount and copy essential configuration like raid, pvm …


    $ ls -alh /dev/mtdblock*
    brwxrwxrwx    1 root     6         31,   0 Oct 14  2002 /dev/mtdblock0
    brwxrwxrwx    1 root     6         31,   1 Oct 14  2002 /dev/mtdblock1
    brwxrwxrwx    1 root     6         31,  10 Oct 14  2002 /dev/mtdblock10
    brwxrwxrwx    1 root     6         31,  11 Oct 14  2002 /dev/mtdblock11
    brwxrwxrwx    1 root     6         31,  12 Oct 14  2002 /dev/mtdblock12
    brwxrwxrwx    1 root     6         31,  13 Oct 14  2002 /dev/mtdblock13
    brwxrwxrwx    1 root     6         31,  14 Oct 14  2002 /dev/mtdblock14
    brwxrwxrwx    1 root     6         31,  15 Oct 14  2002 /dev/mtdblock15
    brwxrwxrwx    1 root     6         31,  16 Oct 14  2002 /dev/mtdblock16
    brwxrwxrwx    1 root     6         31,   2 Oct 14  2002 /dev/mtdblock2
    brwxrwxrwx    1 root     6         31,   3 Oct 14  2002 /dev/mtdblock3
    brwxrwxrwx    1 root     6         31,   4 Oct 14  2002 /dev/mtdblock4
    brwxrwxrwx    1 root     6         31,   5 Oct 14  2002 /dev/mtdblock5
    brwxrwxrwx    1 root     6         31,   6 Oct 14  2002 /dev/mtdblock6
    brwxrwxrwx    1 root     6         31,   7 Oct 14  2002 /dev/mtdblock7
    brwxrwxrwx    1 root     6         31,   8 Oct 14  2002 /dev/mtdblock8
    brwxrwxrwx    1 root     6         31,   9 Oct 14  2002 /dev/mtdblock9

Mount data from **/dev/mtdblock7**

    /bin/mount -t jffs2 /dev/mtdblock7 /data
    # locate data
    $ mount
    /dev/ram0 on / type ext2 (rw)
    /dev/mtdblock7 on /data type jffs2 (rw)
    $ cd /data/
    $ ls
    etc  log  lvm  usr
    $ cd etc/
    $ ls
    abnormal       fstab          localtime      passwd.dav     timezone
    config         hosts          nsswitch.conf  raid.conf
    crontab        kdc.conf       ntpc.conf      resolv.conf
    domain         krb5.conf      pam.d          server
    exports        language.conf  passwd         shadow
    $ cat raid.conf
    0=/VOLUME1|RAID-1989846|vg001|/dev/vg001/lv001
   

You can backup all the files …

## Create your own NFS server
To dump faulty ram disk to external storage, we will create a small chroot with nfs access.
For my part i used Hyper-V from Microsoft with latest debian DVD image, your server need to be on the same network.
![Network](https://i.postimg.cc/P55n7zYB/2019-05-26-09-08-10-Microsoft-Edge.png)

Here are the command to install a small nfs server :

    apt install nfs-kernel-server
    mkdir /root/debian-rootfs
    echo "/root/debian-rootfs *(rw,nohide,insecure,no_subtree_check,no_root_squash,async)" >> /etc/exports
    service nfs-kernel-server restart

Try to access with another computer :

    \\192.168.1.30 (ip of your vm)

![Access fs](https://i.postimg.cc/kXfgD5cP/2019-05-26-09-13-01-192-168-1-30.png)

## Create your own Ramdisk from fresh chroot
It's time to build your own filesystem, for that we will use wheezy because it's ppc compliant and quemu-ppc :)

    # fs installation
    debootstrap --arch=powerpc --foreign --variant=minbase wheezy /root/debian-rootfs http://archive.debian.org/debian/
    
    # Chroot with quemu
    cp /usr/bin/qemu-ppc-static /root/debian-rootfs/usr/bin/ 
    chroot /root/debian-rootfs
    ./debootstrap/debootstrap --second-stage
    
    echo "deb http://archive.debian.org/debian/ wheezy main" > /etc/apt/sources.list
    apt-get update
    
    # Console Configuration
    mknod dev/ttyS0 c 4 64 
    mknod dev/console c 5 1
    
    touch /etc/inittab
    echo "id:2:initdefault:" > /etc/inittab
    echo "si::sysinit:/etc/init.d/rcS" >> /etc/inittab
    echo "~~:S:wait:/sbin/sulogin" >> /etc/inittab
    echo "l0:0:wait:/etc/init.d/rc 0" >> /etc/inittab
    echo "l1:1:wait:/etc/init.d/rc 1" >> /etc/inittab
    echo "l2:2:wait:/etc/init.d/rc 2" >> /etc/inittab
    echo "l3:3:wait:/etc/init.d/rc 3" >> /etc/inittab
    echo "l4:4:wait:/etc/init.d/rc 4" >> /etc/inittab
    echo "l5:5:wait:/etc/init.d/rc 5" >> /etc/inittab
    echo "l6:6:wait:/etc/init.d/rc 6" >> /etc/inittab
    echo "z6:6:respawn:/sbin/sulogin" >> /etc/inittab
    echo "ca:12345:ctrlaltdel:/sbin/shutdown -t1 -a -r now" >> /etc/inittab
    echo "pf::powerwait:/etc/init.d/powerfail start" >> /etc/inittab
    echo "pn::powerfailnow:/etc/init.d/powerfail now" >> /etc/inittab
    echo "po::powerokwait:/etc/init.d/powerfail stop" >> /etc/inittab
    echo "T0:23:respawn:/sbin/getty -L ttyS0 115200 vt100" >> /etc/inittab
    
    # Change password
    passwd 
    exit
    
Boot your new filesytem with uboot : (restart your nas to uboot cmdline)

    setenv serverip 192.168.1.30
    setenv rootpath /root/debian-rootfs
    setenv addtty ${addtty} ${mtdparts}
    setenv nfsargs setenv bootargs root=/dev/nfs rw nfsroot=${serverip}:${rootpath},v3
    run nfsboot
    
Dump your faulty ramdisk on started uboot nfs filesystem

    # install udev
    apt-get install udev
    reboot
    
    #restart to debian fs
    setenv rootpath /root/debian-rootfs
    setenv addtty ${addtty} ${mtdparts}
    setenv nfsargs setenv bootargs root=/dev/nfs rw nfsroot=${serverip}:${rootpath},v3
    run nfsboot

    # dump nand
    dd if=/dev/mtd5ro of=faulty-rootfs.img.gz bs=64 skip=1
  
  Use winrar to extract **ramdisk.bin** (or any other tool who can save corrupt file)
  
![nand](https://i.postimg.cc/GtBPSfzX/2019-05-26-09-41-12-stock-rootfs-img-gz-Version-d-valuation.png)


We are lucky we can list files in this ramdisk corruption is still present but maybe we can directly inject the file system ?

![fs](https://i.postimg.cc/JzBr45YS/2019-05-26-09-47-28-tmp-share.png)

you can find all modules in **/var/lib/modules**

![driver](https://i.postimg.cc/fRbHxHQg/2019-05-26-09-50-22-Microsoft-Edge.png)

## Boot recovered Ramdisk !
In the small debian nfs server, we can decompress the ramdisk and ask uboot to use it

    mkdir /root/stock-ramdisk
    mkdir /mnt/stock-rootfs
    mount -t ext2 -o loop,ro ramdisk.img /mnt/stock-rootfs
    rsync -a /mnt/stock-rootfs/ /root/stock-ramdisk
    echo "/root/stock-ramdisk *(rw,nohide,insecure,no_subtree_check,no_root_squash,async)" >> /etc/exports
    service nfs-kernel-server restart

Restart the nas and from uboot cmdline

    setenv rootpath /root/stock-ramdisk
    setenv addtty ${addtty} ${mtdparts}
    
    setenv nfsargs setenv bootargs root=/dev/nfs rw nfsroot=${serverip}:${rootpath},v3
    run nfsboot

Back to Business :!!:

    
    ...
    Swap Memory On...LED => 1
    Checking File System...0, 0, 0, 0, 0,
    0, 0
    1232739208
    28,33,21,23,0,109,5,22,0
    
    0:0: [sda] Assuming drive cache: write through
    sd 1:0:0:0: [sda] Attached SCSI removable disk
    Starting SMB...
    Starting FTP...
    
    warning: `proftpd' uses 32-bit capabilities (legacy support in use)
    Starting quota...
    Starting Domain Integrate...
    Starting alert agent...
    I2 Event Daemon, Ver 1.0.0.0
    Checking last shutdown...
    Check Data Partition : OK
    
    KARGO-4 R2.0 A1 (Version 02.01.4300.19) - Promise Technology, INC.
    kargo-4-790754 login:
    
    
    
   ## Credz
 
 - Special Thanks to my friend Hikage (f****@!! network cable)
 - Mr.C for serial / ttl usb
 - https://saturn.ffzg.hr/rot13/index.cgi?promise_smartstor
 - https://github.com/alexeicolin/javelin#nfs-boot-first-stage
 - http://jamie.lentin.co.uk/devices/dlink-dns325/replacing-firmware/
 - promise / memup for not hosting their original firmware …
