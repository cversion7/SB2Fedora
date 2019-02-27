First off, besides installing Fedora 29, will be to update the kernel with drivers.

I found that someone had posted similar instructions for Fedora 28 and building Jakeday's kernel modules,
but I wasn't certain how to do that on my own. Unfortunately, the author took down his Github. Thanks to Google cached pages,
I was able to find a cached copy of the instructions.

https://denzilferreira.github.io/microsoft-surface-pro-4-and-fedora-28/

## Getting kernel sources: linux-surface + linux-stable

Jake Day has done all the hard work for us: here. We'll start by creating a folder to host the source code we will need, one for Jake's patches, one for Linux kernel source. If any of the commands do not work, you may be missing some packages installed on your machine.

```
$ cd ~
$ mkdir kernel-sources
$ cd kernel-sources
$ git clone https://github.com/jakeday/linux-surface.git
```

This will download the list of patches from Jake Day's GitHub repository to make changes to the official Linux kernel. Now we'll use Jake's script to install a few libraries and drivers that will be needed for the kernel compilation to suceed.

```
$ cd linux-surface
$ sudo sh setup.sh
$ [enter your sudo password]
$ Running fedora version 28 (Workstation Edition) on a Surface Pro.\n
Press enter if this is correct, or CTRL-C to cancel - enter
-- it will complain about a missing folder, ignore it --
$ Do you want to replace suspend with hibernate (type yes or no) - no
$ Do you want use the patched libwacom packages? (type yes or no) - no
## Additional options appeared for me during this spot, likely due to updates from jakeday
$ Do you want to remove the example intel xorg config? (type yes or no) - I went with no
$ Do you want to remove the example pule audio config files? (type yes or no) - Again, I went with no
-- it will now install IPTS firmware (this will give us touch and pen input!) --
-- it will install motherboard firmware (i915) --
-- it will install wifi Marvell firmware (to fix a buggy WiFi connection) --
$ Do you want to set your clock to local time instead of UTC? This fixes issues when dual booting with Windows. (type yes or no) - no (I'm getting rid of it)
$ Do you want this script to download and install the latest kernel for you? (type yes or no) - no (because this one is meant for Ubuntu and won't work in Fedora)
```

Now we have the firmware in place. Let's now prepare the Linux source to use Jake Day's patches. Check here what is the current supported version of the kernel for which Jake Day has made his patches. At this time it's 4.18.11-1. Remember this version number 4.18.11. Let's now get Linux kernel source code:

### As of writing this (20190227), the latest version is 4.19.23. Commands below updated. Also, cloning the kernel takes a while.

```
$ cd ~/kernel-sources
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
$ cd linux-stable
$ git checkout v4.19.23
```

## Installing packages needed for source code compilation

Before we get compiling everything, we need to make sure we have all the libraries we'll ever need to compile drivers and kernel from source.

```
$ sudo dnf groupinstall "Development Tools" "C Development Tools and Libraries"
$ sudo dnf install elfutils-devel openssl-devel perl-devel perl-generators pesign ncurses-devel
```

## Adding support for battery status monitoring

To add support to battery events, you'll need to apply the patches from qzed which add battery monitoring to your desktop. You'll need to do:

```
$ cd ~/kernel-sources/linux-surface
$ git remote add upstream https://github.com/qzed/linux-surface.git
$ git fetch upstream
$ git checkout master
$ git merge upstream/master
```

## Apply patches from linux-surface into linux-stable

We'll now apply the changes Jake Day has prepared into linux-stable (official Linux kernel). Then we'll load Fedora's configuration file so we have the same support as officially was bundled.

```
$ cd ~/kernel-sources/linux-stable
$ for i in ../linux-surface/patches/4.19/*.patch; do patch -p1 < $i; done
$ cp /boot/config-`uname -r` .config
```

We need to make sure these lines exist in .config file:
CONFIG_SERIAL_DEV_BUS=y
CONFIG_SERIAL_DEV_CTRL_TTYPORT=y

    In the following step "Compiling kernel and modules", make sure to select "m" in the CONFIG_INTEL_IPTS otherwise you'll have errors creating the bzImage. The "m" stands for module, so that's fine!

## Compiling kernel and modules

This will take some time to complete (bzImage about 20 minutes) + (all modules about 45 minutes). Be patient. It will ask you to accept the new modules from Surface Pro 4, shown as (NEW). Press m to have them added as modules for your kernel.

`$ make -j `getconf _NPROCESSORS_ONLN` bzImage; make -j `getconf _NPROCESSORS_ONLN` modules
Installing kernel and modules`

### Lots of m and y (y on ones that m is not an option for). Also, was asked how many CPUs. Chose 4 on i7-8650U (SB2) and all 8 threads stayed above 80% usage almost constantly and most were near or at 100%.

Assuming compilation went without errors, you can now install them both.

```
$ sudo make -j `getconf _NPROCESSORS_ONLN` modules_install
$ sudo make -j `getconf _NPROCESSORS_ONLN` install
```

## Install battery ACPI module

Install DKMS in Fedora:

`$ sudo dnf install dkms`

We'll now download and compile the extra ACPI module (thanks to qzed):

```
$ cd ~/kernel-sources/linux-stable/drivers
$ git clone https://github.com/qzed/linux-surfacegen5-acpi-notify.git surfacegen5-acpi-notify
$ cd surfacegen5-acpi-notify
$ make ##I got ana error here that "No targets specified and no makefile found. Stop." Could not continue.
$ sudo make dkms-install
```

### Ended up after all this with an error that the kernel built has an invalid signature which means I need to sign the kernel thanks to using Secure Boot. Referring to jakeday's article on this: https://github.com/jakeday/linux-surface/blob/master/SIGNING.md

SBSIGN must be installed to do this:
`dnf install sbsigntools`

Updating Grub on Fedora:
`grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg`

Used grub-customizer to set and confirm boot order of kernels: `sudo dnf install grub-customizer`

After all this is done, go back and delete the 'kernel-sources' folder under Home.
