# mkzrx - make zfs root xenial

Ubuntu 16.04 LTS ("Xenial") has builtin support for ZFS-on-root.  However, the
installer does not support it.  There are some [great
instructions](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-16.04-Root-on-ZFS)
on how to do it but it's not for the faint of heart.  This repo contains
scripts to mostly automate the process.

I recommend taking a look at the instructions if you want to better understand
what we're doing or if you have problems with the scripts.

There are two noteworthy things I do differently from that guide.  First, the
above guide explains how to partition using either GPT or UEFI, neither of
which is supported by older motherboards, including my 2012 model.  For greater
compatability I use regular old DOS MBR partitions.

Second, there's a bug in the current Grub which can be worked around by
symlinking the disk device into /dev, so I do that.

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
your diverse hosts using nothing more than zfs send.

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

## Warning

If you run these scripts on a machine with data you care about, you might lose it.  I've tested them extensively
but there's no reason to assume I found all the bugs.

There is a safe way to run this: from an [Ubuntu Live CD or USB key](https://help.ubuntu.com/community/LiveCD).
You want the [64-bit Ubuntu 16.04 Xenial Live CD](http://releases.ubuntu.com/16.04/ubuntu-16.04-desktop-amd64.iso).

Boot it on a machine with no disks containing data you would mind losing.

Note: there's a bug at present whereby the livecd doesn't boot up on its own.
You get dropped to a black screen saying

    Missing parameter in configuration file. Keyword: path
    gfxboot.c32: not a COM32R image
    boot:

Just type "live" (no quotes) and hit enter.

Follow Step 1 from the
[guide](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-16.04-Root-on-ZFS) on
which this is based.  Then scp these scripts onto the target machine.

Alternatively, if you like to live dangerously, you can run them directly on an
existing Ubuntu 16.04 installation, e.g. if you already did a non-ZFS install.
In which case: *BACKUP YOUR DATA* first, to another machine.

## Usage

From the livecd environment, after doing Step 1:

    ./mkzpool zal2 /dev/disk/by-id/usb-SanDisk_Extreme_AA010607160347510209-0:0
    ./mkubuntu zal2 /dev/disk/by-id/usb-SanDisk_Extreme_AA010607160347510209-0:0
    ./mkgrub zal2 /dev/disk/by-id/usb-SanDisk_Extreme_AA010607160347510209-0:0

Set your hostname:

    echo zal > /mnt/etc/hostname 

Your root password is set to 'temprootpw', you probably want to change that:

    chroot /mnt passwd root

Optionally:

    ( echo auto eth0; echo iface eth0 inet dhcp ) > /mnt/etc/network/interfaces.d/eth0

This may not work for you depending on your hardware;in my case I had to use enp1s0 
instead of eth0.  You may prefer to skip this step and just manually run dhclient upon
reboot as described below.  The guide points out that if you're running Ubuntu Desktop
(rather than Server) you should be using NetworkManager and thus you'll want to remove
this file after the reboot.

Now export:

    zpool export zal2

Not strictly required, but highly recommended: follow "Step 6: First Boot" from
the [guide](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-16.04-Root-on-ZFS),
starting with "6.4 Reboot".  You'll need to setup a user account sooner or
later anyway.

It may not come up on the network, especially if you skipped the optional step to
create /etc/network/interfaces.d/eth0.  In that case, login to the console and run

    /sbin/ifconfig -a

Note the name of the interface other than lo0.  Supposing it's enp1s0 as in my case, run

    dhclient enp1s0

Once you have network connectivity, follow "Step 8: Full Software Installation" from the
[guide](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-16.04-Root-on-ZFS).  Optionally do Step 9
as well.  Note that mkubuntu creates several extra snapshots besides the ones the guide does.
I don't see much value in removing them, but it's your call.  In fact I like to do another snapshot
right after step 8:

    zfs snapshot zal2/ROOT/ubuntu@install-desktop

I also suggest after installing ubuntu-desktop but before you reboot that you might want to run:

    zfs create -o canmount=off zal2/var/lib/lightdm
    zfs create -o com.sun:auto-snapshot=false -o org.complete.simplesnap:exclude=on zal2/var/lib/lightdm/.cache 
    chown lightdm:lightdm zal2/var/lib/lightdm/.cache

I found it was another directory like /var/lib/NetworkManager and /var/backups that showed a lot of
activity in my regular snapshots, and that I wanted to exclude from them.

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
that when travelling if something goes wrong my harddisk I have a fallback.  

Example of the send process:

    zfs send -R zal1/ROOT/ubuntu@install-desktop  | zfs recv -d zal2

