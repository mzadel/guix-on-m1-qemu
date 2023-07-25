
# Building an aarch64 Guix VM using qemu on an M1 Mac

Mark Zadel, July 2023

One day, I decided on a whim to try Guix in a VM on an M1 Mac.  It seemed like
a good idea at the time, but I'd never used Guix before and it ended up being
quite a grind to figure out how to get it running.

The initial issue for me was that Guix doesn't provide iso installer images for
aarch64.  All good, you can build one yourself.  After I'd put in the time to
figure out how to do that, though, the iso installer would just hang.  The
serial console would say

    waiting for partition '31393730-3031-3031-3139-333333313833' to appear...

repeatedly.  Apparently [I'm not the only one to see this
behaviour.](https://lists.gnu.org/archive/html/bug-guix/2019-08/msg00134.html)

After much trial and error and futzing around, I learned that, for me, it works
to

 - create a VM image directly from the Guix package manager, or to
 - perform a manual Guix install onto a qemu disk.

[Nathan Henrie has a nice article detailing a lot of relevant information about
installing Guix on M1
Macs.](https://n8henrie.com/2022/10/guix-system-guixsd-vm-on-an-m1-mac/) It
really helped with getting my head into the process.  Thanks!

## Takeaways

The Guix ISO installer didn't work for me on aarch64.  Making a VM image or
doing a manual install did work.

I show a list of steps for getting Guix installed on an aarch64 VM on macOS,
and include a small starter confguration file.

It can be helpful to pull a specific commit of the distro that includes a
working, precompiled kernel.

## Steps to create a Guix aarch64 VM

To create the VM image, we run `guix system image --image-type=qcow2`.  In
order to do that we need to set up a separate linux system to work from
initially.

1. Install qemu via [homebrew](https://brew.sh/):

        user@macos:~$ brew install qemu

1. Get a [Debian arm64 install iso image](https://cdimage.debian.org/debian-cd/current/arm64/iso-cd/).  (arm64 is the same thing as aarch64.)

    You could also use your own favourite distro for this.

1. Create a qemu disk for the Debian install

        user@macos:~$ qemu-img create -f qcow2 debian.qcow2 50G

1. Install Debian onto the new disk in a qemu VM

        user@macos:~$ qemu-system-aarch64 \
            -M virt,highmem=on \
            -accel hvf \
            -m 8G \
            -drive file=debian.qcow2,media=disk,if=virtio,format=qcow2 \
            -drive file=debian-12.0.0-arm64-netinst.iso,id=cdrom,if=none,media=cdrom \
            -device virtio-scsi-device \
            -device scsi-cd,drive=cdrom \
            -bios /opt/homebrew/Cellar/qemu/8.0.2/share/qemu/edk2-aarch64-code.fd \
            -cpu host \
            -device virtio-net,netdev=vmnic \
            -netdev user,id=vmnic \
            -device virtio-gpu-pci \
            -display default,show-cursor=on \
            -device qemu-xhci \
            -device usb-kbd \
            -device usb-tablet

    It might be that not all of these qemu options are necessary.  You might
    have to adjust some of the paths depending on the iso you downloaded and
    which version of qemu you have.

    Follow the Debian installer instructions to install onto the hard drive at `vda`.

1. The installer will reboot you into Debian.

    The boot process might go back to the installer CD instead of the hard
    drive.  (For some reason I often haven't been able to get my Debian VMs to
    boot straight off the hard drive.)  You can manually choose to boot from
    the hard drive in the BIOS:

     - Press escape immediately when the BIOS comes up
     - Go to *Boot Maintenance Manager -> Boot From File*
     - choose the second entry (the hard drive)
     - Then go to *EFI -> debian -> grubaa64.efi*
     - Debian will boot


1. Install the Guix package manager in the Debian system:

        user@debian:~$ sudo apt install guix

1. Update the Guix package descriptions and basic packages:

        user@debian:~$ guix pull

    This will take a while.

1. Create a Guix configuration file on the Debian system.  It defines how your
Guix install is set up and what it will initially contain.

    Put the following into a file called `config.scm` on Debian:

        (use-modules (gnu))
        (use-service-modules networking)
        (use-package-modules certs)

        (operating-system
          (host-name "guix")
          (timezone "Europe/Berlin")
          (locale "en_US.utf8")

          (bootloader (bootloader-configuration
                        (bootloader grub-efi-bootloader)
                        (targets '("/boot/efi"))))
          (kernel-arguments (list "console=ttyS0,115200"))
          (file-systems (append
                  (list (file-system
                                  (device "/dev/vda2")
                                  (mount-point "/")
                                  (type "ext4"))
                                (file-system
                                  (device "/dev/vda1")
                                  (mount-point "/boot/efi")
                                  (type "vfat")))
                              %base-file-systems))

          (users (cons (user-account
                        (name "yourname")
                        (group "users")

                        (supplementary-groups '("wheel"
                                                "audio" "video")))
                       %base-user-accounts))

          (packages (append (list nss-certs) %base-packages))

          (services (append (list (service dhcp-client-service-type))
                            %base-services)))


    This was adapted from the Guix `bare-bones.tmpl` and
    `lightweight-desktop.tmpl` files.

1. Build a Guix VM image.  This will be the qemu hard drive for your Guix VM.

        user@debian:~$ guix system image --image-type=qcow2 --image-size=50G config.scm

    This will take a while.

    At the end of the process, it will leave you with the name of a qcow2 image
    file:

        /gnu/store/z08hk66ig6dn32ivvysphr0d2b0alym0-image.qcow2

    Mine was 422Mb in size.

1. Copy the qcow2 image off the Debian VM, in whatever way you like to do that
(`scp`, attached disk image that's readable in macos, etc.).

1. Shut down the Debian system.

1. Rename the image file on macos to `guix.qcow2`.

1. Start a new VM with the freshly-installed Guix system:

        user@macos:~$ qemu-system-aarch64 \
            -M virt,highmem=on \
            -accel hvf \
            -m 8G \
            -drive file=guix.qcow2,media=disk,if=virtio,format=qcow2 \
            -device virtio-scsi-device \
            -bios /opt/homebrew/Cellar/qemu/8.0.2/share/qemu/edk2-aarch64-code.fd \
            -cpu host \
            -smp 4 \
            -device virtio-net,netdev=vmnic \
            -netdev user,id=vmnic \
            -device virtio-gpu-pci \
            -display default,show-cursor=on \
            -device qemu-xhci \
            -device usb-kbd \
            -device usb-tablet

1. Log into guix with the root user, which has an no password by default.

1. Set a password for root and for your user account

        root@guix ~# passwd
        ...type in the new password...
        root@guix ~# passwd yourname
        ...type in the new password...

1. Log out of root in the guix vm and log back in as the user account

1. Update the Guix package descriptions

        yourname@guix ~$ guix pull

1. Copy the `config.scm` you created above onto your Guix system

    As above: you can do this via `scp`, an attached disk image that's
    readable in macos, etc.

    You can install `ssh` and `scp` with `guix install openssh`.

1. Copy the config file to `/etc`.

        yourname@guix ~$ sudo cp config.scm /etc

    `/etc/config.scm` is just a convenient place to keep your configuration as
    you work with the system.

1. Make the `/boot/efi` directory and mount the EFI parition there

        yourname@guix ~$ sudo mkdir /boot/efi
        yourname@guix ~$ sudo mount /dev/vda1 /boot/efi

1. Reconfigure the Guix system:

        yourname@guix ~$ sudo guix system reconfigure /etc/config.scm

1. Reboot the Guix system

        yourname@guix ~$ sudo reboot

1. Nice one!  Enjoy your new Guix system.

    Check out [the Guix
    documentation](https://guix.gnu.org/en/manual/en/html_node/Getting-Started.html)
    for next steps.

## Pulling a specific commit

Since Guix is a rolling release distro, there are sometimes broken packages and
long compile times, especially on aarch64.  This isn't really a problem since
upgrades are atomic, but it can be confusing/frustrating to a newcomer.  In
general, certain packages might have errors or might not have binary downloads
(substitutes) available.

I used a `--commit` argument in the `guix pull` commands above to be sure to
get `linux-libre-6.3.13`.

        yourname@guix ~$ guix pull --commit 00ed2901f5

At the time of writing, the [6.4 version hasn't been built on
aarch64,](https://ci.guix.gnu.org/search?query=linux-libre-6.4+spec%3Amaster+)
so if you use the latest Guix version you'll get into a long upgrade situation
where it rebuilds the kernel.  I solved it by using a slightly older Guix
([`00ed2901f5`](https://git.savannah.gnu.org/cgit/guix.git/commit/?id=00ed2901f5)).
I haven't been able to figure out another method to pin the install at a
particular kernel version in my `operating-system` definition.

## Manual Installation

The other method that I got to work was a manual installation.  You create a
qemu disk, partition and format it, and then use `guix system init` to install
to it.

This is basically just the [guix manual installation
instructions](https://guix.gnu.org/en/manual/en/html_node/Manual-Installation.html),
spelled out for the case of an aarch64 installation in a qemu VM.

1. Create a new disk for your Guix installation:

        user@macos:~$ qemu-img create -f qcow2 guix.qcow2 50G

1. Follow the above directions for getting a Debian VM set up.

1. Boot into your Debian system with the new, blank guix disk attached.  Add
the following to the initial `qemu-system-aarch64` command and run it:

        -drive file=guix.qcow2,media=disk,if=virtio,format=qcow2 \

    You might need to manually choose the disk to boot from in the BIOS.

1. Install the Guix package manager in Debian with `sudo apt install guix`

1. Run `guix pull` as the normal user account:

        user@debian:~$ guix pull

1. Run `cfdisk` or `gparted` as root in the Debian VM to `vdb`, which will be
your guix disk.

        root@debian:~# cfdisk /dev/vdb

    Create a GPT partition table.

    Create the following partitions:

    - A 1Gb partition with type 'EFI System'
    - A 49Gb partition with type 'Linux filesystem'

1. Prepare the filesystems on `vdb`

        root@debian:~# mkfs.vfat /dev/vdb1
        root@debian:~# mkfs.ext4 /dev/vdb2

    If you used `gparted` above, you don't need this step since it makes the
    filesystems itself.

1. Mount the `vdb` partition and make an `etc` directory:

        root@debian:~# mount /dev/vdb2 /mnt/
        root@debian:~# mkdir /mnt/etc

1. Mount the `vdb` EFI boot partition under `/mnt/boot/efi/`

        root@debian:~# mkdir -p /mnt/boot/efi
        root@debian:~# mount /dev/vdb1 /mnt/boot/efi/

1. Copy `config.scm` from above onto the mounted guix disk under `/mnt/etc`

        root@debian:~# cp config.scm /mnt/etc

1. Run guix's manual installation procedure:

        user@debian:~$ sudo guix system init /mnt/etc/config.scm /mnt

    It'll download a bunch of packages and install them in `/mnt`.

1. Shut down Debian and boot into the Guix VM using the `guix.qcow2` disk.

1. Nice one!  Enjoy your new Guix system.

