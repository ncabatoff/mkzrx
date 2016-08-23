# mkzrx - make zfs root xenial

Ubuntu 16.04 LTS ("Xenial") has builtin support for ZFS-on-root.  However, the
installer does not support it.  There are some [great
instructions](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-16.04-Root-on-ZFS)
on how to do it but it's not for the faint of heart.  This repo contains
scripts to mostly automate the process, at least for steps 2-5 and 7.

I recommend taking a look at the instructions if you want to better understand
what we're doing or if you have problems with the scripts.

There are two noteworthy things I do differently from that guide.  First, the
above guide explains how to partition using either GPT or UEFI, neither of
which is supported by older motherboards, including my 2012 model.  For greater
compatability I use regular old DOS MBR partitions.

Second, there's a bug in the current Grub which can be worked around by
symlinking the disk device into /dev, so I do that.

There are also a slew of smaller changes, some of which may break things.  This
is not 100% faithful to Richard Laager's instructions, and odds are any
breakage is my doing and not his.

## Option 1: LiveCD Install Of Minimal Bootable System

Boot from an [Ubuntu Live CD or USB key](https://help.ubuntu.com/community/LiveCD).
You want the [64-bit Ubuntu 16.04 Xenial Live CD](http://releases.ubuntu.com/16.04/ubuntu-16.04-desktop-amd64.iso).

### Danger

Run this on a machine with no disks containing data you would mind losing.  It
should be safe, but if there's a bug in these scripts and it damages your
system, you get to keep both pieces.

### Booting

Note: there's a bug at present whereby the livecd doesn't boot up on its own.
You get dropped to a black screen saying

    Missing parameter in configuration file. Keyword: path
    gfxboot.c32: not a COM32R image
    boot:

Just type "live" (no quotes) and hit enter.

### Running the scripts in the LiveCD Environment

Hit Ctrl-Alt-T to open a terminal.

Follow Step 1 from the
[guide](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-16.04-Root-on-ZFS) on
which this is based.  Then scp or wget these scripts onto the target machine.

In these examples my hostname is zal, and I'm calling this pool zal2.
I'm building it on a USB key, but it could also be an SSD or HDD.

You must remove all partitions on the target device before calling these scripts,
or have exactly one ZFS (type BF) partition that isn't already part of a pool.

As root:

    ./mkzpool zal2 /dev/disk/by-id/usb-SanDisk_Extreme_AA010607160347510209-0:0
    ./mkubuntu zal2 /dev/disk/by-id/usb-SanDisk_Extreme_AA010607160347510209-0:0
    ./mkgrub zal2 /dev/disk/by-id/usb-SanDisk_Extreme_AA010607160347510209-0:0

Set your hostname:

    echo zal > /mnt/etc/hostname 

Your root password is set to 'temprootpw', you probably want to change that:

    chroot /mnt passwd root

### Setup temporary network (optional)

    ( echo auto eth0; echo iface eth0 inet dhcp ) > /mnt/etc/network/interfaces.d/eth0

This may not work for you depending on your hardware; in my case I had to use enp1s0 
instead of eth0.  You may prefer to skip this step and just manually run dhclient upon
reboot as described below.  The guide points out that if you're running Ubuntu Desktop
(rather than Server) you should be using NetworkManager and thus you'll want to remove
this file after the reboot.

### Reboot

Export your new zpool and reboot.

    zpool export zal2
    reboot

Remove the LiveCD disk/usbkey so you boot off your new install.
Continue from "First boot with ZFS" ,below.

## Option 2: Use Vagrant and VirtualBox Minimal Bootable System

Tested with 1.8.5 and [VirtualBox
5.0](https://www.virtualbox.org/wiki/Download_Old_Builds_5_0) (Vagrant 1.8.5 is
the version available in CentOS7 and Ubuntu 16.04, but it doesn't yet support
VirtualBox 5.1).

This is still experimental, do the LiveCD option if you don't want to break
anything.

### Danger

If you run these commands on a machine with data you care about, you might lose
it.  I've tested them extensively but there's no reason to assume I found all
the bugs.  The LiveCD option above is safer but requires more of your attention,
whereas the Vagrant option is largely fire-and-forget.

### Using a pool name other than rpool

As described below under the mkzpool section, I suggest you not name the pool
rpool.  For now this requires editing Vagrantfile; search for 'rpool' and
replace it with your desired poolname.  You'll also have to modify the commands
given in the following section accordingly.

### Running the scripts in Vagrant

First edit Vagrantfile to customize the lines at the top saying

    rpool = 'rpool'
    hostname = 'yourhostname'

(only hostname is strictly required, you can leave rpool alone if you prefer.)

Then from the same directory run

    vagrant up   # wait for it to exit

This will run the three scripts in a virtual environment, installing to a second virtual disk.
You will see man errors in red such as

    ==> default: perl: warning: Setting locale failed.
    ==> default: perl: warning: Please check that your locale settings:
    ==> default: 	LANGUAGE = "en_US:",
    ==> default: 	LC_ALL = (unset),
    ==> default: 	LC_COLLATE = "C",
    ==> default: 	LANG = "en_US"
    ==> default:     are supported and installed on your system.
    ==> default: perl: warning: Falling back to the standard locale ("C").

or

    ==> default: dpkg-preconfigure: unable to re-open stdin: No such file or directory

(among others.)  As far as I know these are harmless.  Upon success the command will
finish with these lines (the numbers may differ):

    ==> default: Generating grub configuration file ...
    ==> default: Found linux image: /boot/vmlinuz-4.4.0-34-generic
    ==> default: Found initrd image: /boot/initrd.img-4.4.0-34-generic
    ==> default: done

Now run

    vagrant halt

to shutdown the VM.  

### Testing

To test the result (i.e. that it can boot off the new ZFS install):

    VAGRANT_VAGRANTFILE=Vagrantfile.deleteOrigDisk vagrant up

This deletes the original ext4 disk provisioned by Vagrant, and boots off the
new zfs-on-root VM.  Vagrant won't be able to work with it because we didn't
persist any of the stuff that Vagrant installs to communicate with the VM.
Instead use the VirtualBox GUI to login as root with password 'temprootpw' and
satisfy yourself that all is in order.  Poweroff the guest using the VirtualBox
GUI, since Vagrant has lost control of the VM (no need to save state).  

### Installing the VirtualBox image on a real disk

Now we need to move the disk image to real device.  You'll need to have zpool
and zfs commands installed, see [the ZFS on Linux](http://zfsonlinux.org/) site
for installation instructions if you haven't already done so.

Since you're not running these commands in the VM, here's the first chance you
have to mess up your existing setup.  

*BE CAREFUL!  MAKE SURE YOU HAVE THE RIGHT TARGET DISK!*

    ./vdiToDisk rpool /dev/disk/by-id/target-disk
   
Boot off /dev/disk/by-id/target-disk and continue immediately below with "First
boot with ZFS".

## First boot with ZFS

Once you've managed to boot off your new minimal ZFS-on-root install, you've still
got some setup to do.

Login as root with password 'temprootpw' or whatever you reset it to.

Not strictly required, but highly recommended: follow "Step 6: First Boot" from
the [guide](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-16.04-Root-on-ZFS),
starting with 6.5 "Wait for the newly installed system to boot normally. Login
as root."  You'll need to setup a user account sooner or later anyway.

If you're not on the network after the reboot, find your desired network interface, e.g. with

    /sbin/ifconfig -a

or

    ip addr

Supposing it's enp1s0 as in my case, run

    dhclient enp1s0

to get yourself a network address via DHCP.

Once you have network connectivity, follow "Step 8: Full Software Installation" from the
[guide](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-16.04-Root-on-ZFS).  Optionally do Step 9
as well.  Note that mkubuntu creates several extra snapshots besides the ones the guide does.
I don't see much value in removing them, but it's your call.  I also like to do another snapshot now:

    zfs snapshot zal2/ROOT/ubuntu@install-desktop

### Docker

In principle there's no reason that /var/lib/docker should be part of the root
dataset.  It too generates lots of activity, at least on my system.  Not a
large volume of data, just a steady stream.  Any data I care about involving
docker is in the form of volumes, which is another fs in my setup
(zal2/data/docker).  

If you make /var/lib/docker a zfs dataset, Docker will stop working.  The easy
solution I found was to enable ZFS storage in Docker:

    root@xal:/etc/default# git diff -r 55d29aa3e2af41f39b925d4b899fc562a7a7f7ff .
    diff --git a/default/docker b/default/docker
    index da23c57..0b0b0c6 100644
    --- a/default/docker
    +++ b/default/docker
    @@ -11,7 +11,7 @@
     #DOCKER="/usr/local/bin/docker"
     
     # Use DOCKER_OPTS to modify the daemon startup options.
    -#DOCKER_OPTS="--dns 8.8.8.8 --dns 8.8.4.4"
    +DOCKER_OPTS="-s zfs"
     
This is the first time I've tried this, I've done almost no testing, so watch out,
but so far so good.

## Other uses

If you preserve the snapshots, you can skip the lengthy mkubuntu step in
future.  For example, in my case once I had done the above process to setup my
desktop, I decided to give my laptop the same treatment.  This required running
mkzpool to partition the SSD and create the zpool, using zfs send to populate
its $pool/ROOT/ubuntu based on my desktop's $pool/ROOT/ubuntu@install-desktop
snapshot, and running mkgrub.  It took a couple of minutes all told.

After that I customized the hostname and updated /etc/fstab to reflect the new
swap device (since I used a different name for the root pool), rebooted, and I
was good to go.

I repeated the same process when I bought a fast USB key to use as a backup, so
that when travelling if something goes wrong with my harddisk I have a fallback.  

Example of this process:

    ./mkzpool zal1 /dev/disk/by-id/ata-ADATA_SX900_7D4120000612 
    zpool import -N -R /mnt zal1
    zfs send -R zal2/ROOT/ubuntu@install-desktop  | zfs recv -d zal1
    zfs send -R zal2/home  | zfs recv -d zal1
    ./mkgrub zal1 /dev/disk/by-id/ata-ADATA_SX900_7D4120000612 
    zpool export zal1

## The scripts

### mkzpool

mkzpool is a simple script to create a root zpool and some container datasets.
It will also partition the disk if there isn't already a partition table.
You might want to pre-create a partition table if you want to reserve some space.
For example, on a USB key it might make sense to save some room for a VFAT partition.
I've only tested the code in this repo with the first partition assigned to ZFS.
If you take this route, the partition must be of type 'bf'.

To run it simply give it the poolname and device name, for example:

    ./mkzpool zal2 /dev/disk/by-id/usb-SanDisk_Extreme_AA010607160347510209-0:0

I suggest not using a generic name like 'rpool', but rather to name it after the
machine's hostname.  This will make it easier to manage if you ever want to have
the disks coexist in the same machine.  On the other hand, the advantage to using
a generic name is that it's one less customization, making it easier to synchronize
your diverse images using nothing more than zfs send.

### mkubuntu

mkubuntu is invoked in the same way as mkzpool:

    ./mkubuntu zal2 /dev/disk/by-id/usb-SanDisk_Extreme_AA010607160347510209-0:0

This does the OS installation onto the pool created by mkzpool.  mkubuntu
creates a minimal bootable ROOT/ubuntu dataset and installs grub.  There are
two interactive bits: you'll have to provide your timezone and tell grub what
disk to install on.  (I'm working on eliminating those dialogs to further
automate the process.)  

### mkgrub

mkgrub is invoked in the same way as mkzpool:

    ./mkgrub zal2 /dev/disk/by-id/usb-SanDisk_Extreme_AA010607160347510209-0:0

This makes the zpool bootable.

