# paczfs

Copyright 2018 Varasys Limited

Install ArchLinux to Block Device or Image File with ZFS Root Filesystem

This script uses the ArchLinux pacstrap utility (and other related utilities) to install ArchLinux to a block device or image file with ZFS root filesystem including an EFI boot partition.

The practical result is that it creates an ArchLinux image which can be booted on bare metal or in a virtual machine to run ArchLinux with a zfs root filesystem.

# Getting Started

To download paczfs from the github master repository:

```
curl -LO https://raw.githubusercontent.com/varasys/paczfs/master/paczfs
```
To create an image using default settings run the following (but don't run it yet, at least read to the end of this section):

```
./paczfs
```
If any dependancies are not installed, you will be prompted for confirmation to install them (use the `-I` option to suppress the prompt).

Not including download time this should take between two and five minutes on an average laptop to create an image file named "paczfs-${SUFFIX}.img" (where the `SUFFIX=$( date +%s)` to act as a sort of timestamp) which will be a 20 GB sparse file (using about 900MB actual space).

A customized image file can be created by specifying a configuration file to do things such as copy ssh keys, change keyboard layouts, create VDI, QCOW, or QCOW2 images or to write the image directly to a block device (such as directly to a hard drive) instead of a file.

To create a default configuration file run (uppercase -C with stdout redirected to a file):

```
./paczfs -C > paczfs.conf
```
The default configuration file is fairly straightforward and has some comments. See below for more detailed information about configuring and customizing the final image.

To create an image using a configuration file run (lowercase -c):

```
./paczfs -c ./paczfs.conf
```

It is also possible to set any variable from the config file directly on the command line. For example, the following creates the default image and then converts it to VDI format so it will run on VirtualBox (which we guess is actually the first command most people will want to run, which is why we asked you to finish reading this whole section). Note that if a variable is defined in the config file it will have precedence over variables specified on the command line, and also that the single quotes prevent the shell from actually evaluating the ${SUFFIX} variable (paczfs will evaluate it during execution so if the final file name is "paczfs-.vdi" double check that you used single quotes).

```
VDI_FILE='./paczfs-${SUFFIX}.vdi' ./paczfs
```
This will produce a .vdi image file that can be booted in VirtualBox (along with the default raw image file).

Most people will probably also want to uncomment and adjust as needed the "copy-ssh-keys" hook function in the default configuration file (and maybe define one of the hook functions to disable root password since it is empty by default). This will make more sense after you look at the default config file (so if you haven't yet, you should now).

Note that paczfs will use default values for variables not defined in the config file. The only special exceptions are if a block device is specified the default is not to create a raw image file (since paczfs can write directly to the block device) and the default size will be equal to the size of the block device (ie. no need for you to set explicitly).

If the CACHE_PATH variable has a colon ":" then `sshfs` will be used to mount it on /var/cache/pacman/pkg (which is `pacman`'s default package cache). The part before the colon will be considered as the server name, and any extra options can be set in the ~/.ssh/config file.

If you haven't used ArchLinux before it is important to understand that this script is very deliberately designed to install the smallest practical (but useful) system possible as a base so you can have total control over anything extra, so lots of what you are used to probably won't be installed by default (but is most likely available).

For example, unless you explicitly install them, the `shutdown` and `reboot` commands aren't available (so use `systemctl poweroff` and `systemctl reboot` instead). It is a bit frustrating at first, but is ultimately very useful because it forces you to understand which tools you are actually using (since sometimes packages try and maintain consistency to use sym-links or other compatibility tricks that obfuscate what is really happening).

The environment variable WORKSTATION_PKGS defines a list of packages we have chosen to add to the image in addition to the ArchLinux base package (things we consider to be core packages). Feel free to customize per your liking.

Note that one of the things included in the default WORKSTATION_PKGS list is the `pkgfile` utility which can be used to find out which ArchLinux package a file is located in (ie. to find out which package to install to get a specific command).

The `paczfs` script takes snapshots during image creation at the following three times.

1. zroot@miniroot - after the base system is installed (but without a kernel)
2. zroot@bootable - after the kernel and bootloader is installed
3. zroot@workstation - after the packages in the `WORKSTATION_PKGS` array is installed

When booting into an image for the first time the "zroot@workstation" snapshot will be active, and you can always get back to this state by cloning this snapshot and booting into the clone as shown in the following example.

```
zfs clone zroot/system/default@workstation zroot/system/fresh_start
zpool set bootfs=zroot/system/fresh_start zroot
systemctl reboot
```
It is also possible to rollback to this state (but probably not on a dataset that is currently mounted so probably easier to clone is described above):

```
zfs -r zroot/system/default@workstation
```

The "miniroot" snapshot can be cloned and used in containers that require systemd. By default systemd looks for machine directories in the /var/lib/machines directory, so the paczfs script creates a zroot/machines directory at that mount location (note that this is a conflict with using btrfs filesystem). For example, the following will create and boot into a clone using the miniroot snapshot.

```
zfs clone zroot/system/default@miniroot zroot/machines/miniroot_clone
systemd-nspawn -b -M miniroot_clone --bind=/root --bind=/var/cache/pacman/pkg
```
We find developing containers using this miniroot and `pacman` much more more intuitive and straightforward than trying to write docker files or work with upstream docker images that don't exactly match our use cases. Basically just write a bash script to install and configure whatever you want in the container. When your are done you can lock it down a little by uninstalling `pacman`. You can share data by bind mounting directories into multiple containers and the `systemd-resolved.service` will include containers in dns lookups (but we always try to prefer unix sockets for inter-container communication since we think securing ip networking is too complex so we just try and avoid it completely as much as possible).

The "bootable" snapshot represents the minimum requirements to boot the machine, and represents a very "locked down" state (since so little is installed). This snapshot can be cloned to use as a minimal root filesystem. To get even more minimalistic you could, and probably should, uninstall `pacman` from production images after installing what you need.

## VERY IMPORTANT
This part is the most important thing to understand in order to understand how to boot into different root filesystems. Remember that this configuration is not well tested at this point (which is why we are trying to provide detailed explanations).

There are a couple ways to configure how and where zfs datasets are mounted and whether they should be mounted according to the /etc/fstab file or using `zfs mount`.

Since not much of the online discussion (that we have seen) focuses on the implications for booting into multiple root filesystems, this deserves a little attention to understand why we implemented it the way we did.

To make things clear we will start with a quick summary of how/when zfs datasets are mounted. But first it is important to agree to some terminology. A "pool" is what is listed by the `zpool list` command, and a "dataset" is what is listed by the `zfs list` command, and a "mountpoint" is a physical directory onto which something can be mounted.

When this script creates the original zfs pool it sets the following property on the pool:

- bootfs=${POOL_NAME}/system/default

and sets the following defaults for all datasets created on the pool:

- mountpoint=legacy
- canmount=noauto

The dataset "mountpoint" property is used to determine whether a dataset should be mounted using the system `mount -t zfs` command or using the zfs `zfs mount` command. In other words whether mounting the dataset is the responsibility of the system (using the /etc/fstab file) or zfs (using scripts such as zfs-mount.service).

If the "mountpoint" property is set to the special word "legacy" then the system `mount -t zfs` command must be used to manually mount the filesystem and it will be automatically mounted on boot if it is listed in the /etc/fstab file. But in this case, the `zfs mount` command can't be used to mount the filesystem and zfs won't be able to automount it (since zfs doesn't know the mountpoint). Also, the "canmount" property is ignored (if a dataset is listed in the /etc/fstab file it will be mounted by the system regardless of the "canmount" property).

On the other hand, if the "mountpoint" property is set to a valid mountpoint, the opposite is true. The dataset can only be mounted with the zfs `zfs mount` command (the system `mount -t zfs` command will raise an error). In this case, the "zfs-mount.service" unit runs the `zfs mount -a` line where the "-a" option mounts all datasets which have the "mountpoint" property set (unless it is set to "legacy") and the "canmount" property set to "on". If "canmount=noauto" the dataset can still be mounted manually using the `zfs mount` command but won't be mounted automatically by the `zfs mount -a` command, and if the "canmount=no" the dataset can't be mounted.

In a typical installation the "zfs-import.target" "zfs-import-cache.service" and "zfs-import-scan.service" systemd unit files are required to import the pools, but in this case, these operations are performed during early boot (as described below), so these these unit files are not enabled by default.

The two other zfs systemd unit files which are not enabled by default are the "zfs-share.service" and "zfs-zed.service", but you can enable them if needed. Note that the "zfs.target" unit is enabled by default.

So, in summary:
- if "mountpoint=legacy" the /etc/fstab file will be used to determine whether/where to mount the dataset at startup
- if the "mountpoint" property is set to a valid path, and "canmount=on" the dataset will be mounted by the "zfs-mount.service" at startup

By default, this script sets the default "mountpoint" property of the pool to "legacy", which means that all zfs datasets inherit it and mounts are performed using the /etc/fstab file in whatever root filesystem was booted into. It is important to understand that if you clone a root filesystem and then boot into it the reason it will have all the same mounts is because the /etc/fstab file was included in the clone. If the clone should have different mounts edit the /etc/fstab file in the clone as required.

On the other hand, if you set the "mountpoint" property of a dataset to a valid path and set "canmount=on" the dataset will be mounted when you boot into any root filesystem that has the "zfs-mount.service" enabled.

Everything described above is valid for all systems with zfs regardless of whether they use a zfs root filesystem. To use zfs as the root filesystem we need to add the zfs kernel modules and startup script to the initram filesystem (which is temporary ram based root filesystem in used to initialize the real root filesystem).

Think about it this way; the zfs kernel modules are stored on the root filesystem, so how can be they be loaded into the kernel before the root filesystem is mounted; but how can the root filesystem be mounted if the kernel doesn't have the zfs kernel drivers?

The solution is for the kernel to first mount a temporary ram based filesystem which includes whatever modules and scripts are required to mount the real root filesystem, and then pivot to the real root filesystem. This temporary filesystem is stored in an image file in the EFI partition which is formatted with fat32 which the kernel can read without special help.

Using an initramfs during bootup is normal and not specific to zfs, but to use zfs as the root filesystem we need to make sure that the kernel modules and zfs initialization script are included correctly in the initramfs image.

The "magic" which actually loads the zfs drivers, imports the zpool, and mounts the selected root filesystem dataset (according to the zpool "bootfs" property) at boot time is the "zfs" script in the hooks directory of the initramfs image created by `mkinitcpio`. This operation is performed in the "prepare-initramfs" section of the script and includes copying the script located at /usr/lib/initcpio/hooks/zfs into the initramfs.

This script uses logic to provide several ways to specify how to select a zfs root filestem using various linux kernel arguments (ie. zfs=auto, zfs=bootfs, root=ZFS=, etc). Type `mkinitcpio --hookhelp zfs` (on a machine that has the mkinitcpio and zfs packages installed) to see the various options that can be appended to the linux command line (in /etc/default/grub).

The default ArchLinux zfs installation tries to use the "root=ZFS=pool/dataset" method, but in the "install-grub" section of the script we use `sed` to change this to the "zfs=bootfs" method. The reason is because we want to be able to use the zpool "bootfs" property to select a root filesystem (instead of having to edit grub configuration files).

The only problem with the "zfs=bootfs" method is that the default initramfs zfs scripts performs  a check to make sure the /etc/fstab on the initramfs contains an entry for each dataset with the "mountpoint" property set to "legacy" before mounting it. Although this is a reasonable check, for our use case it is not appropriate. There is a `sed` command in the "prepare-initramfs" section of the `paczfs` script which modifies the zfs initialization script to short circuit this check (so you won't need to update the initramfs /etc/initfs for each dataset that you want to be able to boot into).

In summary, if you are happy to control everything using the "/etc/fstab" file in each root filesystem everything should already be set-up (just change the zpool "bootfs" property and reboot). If you want to try something else, hopefully this explanation provides a good explanation of the starting point used by the `paczfs` script.

Note that it is important for only one zpool to have the "bootfs" property set, because the default zfs initramfs scripts try to mount the "bootfs" on the first pool detected that has one.

If there is a problem mounting the root filesystem you will drop into a shell with some basic tools you can use to try and troubleshoot the problem (including the `zpool` and `zfs` programs).

# What is the Point (short answer)

The practical benefit of a zfs root filesystem is the ability to securely snapshot, clone, and transfer root filesystems for better management in large clusters enabling a flash A/B type of upgrade to systems and containers on those systems and robust error recovery (ie. snapshot on every successful boot and try snapshots in reverse chronological order for recovery). Basically, to start managing the operating system itself the same way we have been handling our data with zfs.

Certainly there are other ways to achieve this, which is why we are releasing this script to solicit feedback on the long term feasibility of this strategy, and to learn about alternatives. So please raise issues, comments, requests and all other feedback on the github site.

This script started simple, but evolved in a way that embraced the strategy that the best compromise between simplicity and customizability is to think about the user experience from two different perspectives:

1. On one level it should be as simple to use as a Dockerfile
2. If the user knows how to do something; this script should not be a limitation to implementing it no matter how complicated it is

Our solution (which we borrowed from so many before us) was to use bash and implement hook functions which the end user can define in a config file and which can execute arbitrarily complex operations, and optionally override the pre-defined operations. It seemed to us that every tool we were trying to use would have some limitation which we would ultimately end up being simple in bash, but cumbersome to get to bash. Eventually we realized it was easier to just use bash in the first place.

This means that there are two ways to look at the script when you open it up:

1. You want to find out what the pre-defined operations are so you can write a hook function to make a customization
  
  If this is what you are doing you should only consider the "initialize" and "create" image functions. These functions work in a very linear step-wise process showing each step of the process and try to avoid "bashfu" ("bashfu" meaning special bash tricks)

2. You are trying to debug or change the actual script

  We realize there aren't as many comments as there could be. This is the practical result of the script changing to much before we released it to keep the comments accurate. Also, bash has some nuances which are hard to describe in words (to us at least) but obvious in the bash syntax (if you know what to look for). During development it is easier for us to not include comments (we waste too many hours following stale comments). But to the extent that there is somebody out there to read them, we are willing to write them.
  
  So the best way to approach this is to raise an issue on github and let us point you in the right direction (we are really keen to develop a community to test the feasibility of ArchLinux/ZFS system management so we will be supportive!).

  Even better if you can bring some technical knowledge (such as getting BIOS booting working; which we don't use, so it's not our priority, but if someone will tell us which three lines to add we will find where to add them).

## CONFIGURATION
If a configuration file is provided it is sourced before script execution. This has the same effect as literally cutting and pasting the config file into the script. The point is that: 1) the config file uses bash syntax, and 2) the config file is sourced (or included into this script) during execution.

The config file is used to set environment variables and define "hook functions", which are user defined functions which are run at pre-determined times throughout the image creation (see below).

The command line options to paczfs only control debug and configuration settings. Environment variables are used to configure the final image. The environment variables may be set on the command line prior to executing `paczfs`, or can be set in the config file (initially created with `paczfs -C > out_file`). The priority of variables is the config file first, command line second, and defaults third.

In addition to defining environment variables, hook functions may be defined in the configuration file. "Hook functions" are functions which the script will execute at pre-determined stages during the image generation if they are defined in the config file. The default configuration template has some examples (commented out) of hook functions to copy ssh keys and set the default keyboard layout.

Run `paczfs -C` to see a configuration file template listing the applicable environment variables and hook functions. Uncomment and adjust the variables and hook functions as needed.

Note that paczfs will "evaluate" each variable (using eval), so it is okay to "embed" other variables in single quotes (ie. VDI_FILE='${SUFFIX}.vdi'). The single quotes will prevent the shell from expanding the variable (but paczfs will still expand it when needed, which is typically after the referenced variable has been set).

To see when each hook function will be executed scan the script for lines that say 'if fire-hook ' which will fire the hook listed in the if statement. You can prevent the pre-defined code from executing by setting "handled=true" in the hook function (in which case you are responsible for ensuring that everything that is required in that section is performed in your hook function).

When exploring the main script to find the best hook function to inject your own scripting focus on the "initialize" and "create-image" functions starting with the first "if fire-hook" line.

## CONTAMINATION
paczfs tries to minimize impact to the host system, but specifically aims to create identical images given the same config file (not necessarily bitwise equal, but functionally equivalent), so it is careful to not copy things like the current system locale into the new image. This is an aspirational goal at this point and has not been confirmed. The reason paczfs tries to be careful not to contaminate the new image is so the user has a fully defined starting point for using hook functions to specifically customize as needed with no surprises (ie. the same image result no matter how the host computer is set up).

One exception to this philosophy of keeping the image "clean" is that this script and the configuration file (if provided) will be saved to the /etc/paczfs directory on the newly created image and all packages listed in the `WORKSTATION_PKGS` array will be installed.

There are a couple more conveniences such as copying files from /etc/skel into /root, setting the timezone to UTC, and starting systemd-networkd/systemd-resolved (but note that none of these things are linked in any way to the host system).

## ROOT FILESYSTEM
Paradoxically, by default, the zfs systemd unit files are not enabled because the kernel modules are loaded and datesets mounted very early in the boot stage (before the systemd unit files are even available) so they aren't needed. Be aware of this if you have a use case that requires them to be enabled (in case the results aren't what you expected). Also note that this means the zfs evend daemen service is not enabled (we still haven't gotten around to investigating what this does), so enable it manually if you need it (and then open an issue at github to tell us what you use it for).

Because the root filesystem in on ZFS, you can snapshot and clone the root filesystem and reboot into any clone instantly (due to ZFS Copy On Write (COW) functionality). But perhaps more importantly, you can update images using zfs send and receive functionality.

Set the zpool "bootfs" property (ie. `zpool set bootfs=zroot/dataset zroot`) to select a dataset to mount on / (as the root filesystem) on the next boot. 

```
zfs snapshot zroot/system/default@clone # create a snapshot (you can only clone snapshots)
zfs clone zroot/system/default@clone zroot/system/clone # clone the snapshot
zpool set "bootfs=zroot/system/clone" zroot # set the "bootfs" property on the pool to the clone
```
If you clone the current root filesystem and then set "bootfs" to the clone then all of the filesystems that are currently mounted will be mounted when you reboot with the clone mounted as the root filesystem. This is because the zpool is created with the "mountpoint" option set to "legacy" which disables automatic mounting and unmounting by zfs (it is handled by the "legacy" /etc/fstab file).

By default, the zpool "mountpoint" is set to legacy, so zfs mounts are handled by the /etc/fstab file instead of zfs. This is a subtle but important point. The root filesystem should always be set to legacy and will will be mounted during early boot along with all datasets which have a mountpoint defined and also have the "canmount" property set to "on". Generally this is not what we want. By setting the "mountpoint" property to legacy mounting the filesystem is completely controlled by the /etc/fstab file and not zfs (all possible root datasets should always have the "mountpoint" property set to legacy.

Another possability is to set the zfs mountpoint property to a real path but set the "canmount" property to "noauto". In this case, the dataset will never be automatically mounted, but can still be mounted with the `zfs mount` command manually.

The guidance above focuses on how to switch the root filesystems and then select what else gets mounted selectively based on which root filesystem was selected (ie. by referring to the /etc/fstab on whatever was selected). To achieve the opposite effect of making sure that a zfs dataset is mounted no matter which root filesystem is selected you can set both the "mountpoint" property to a valid path, and "canmount=on" and they will be mounted during the early boot process immediately after the root filesystem is mounted (regardless of which root filesystem is selected).


In theory, but not yet extensively in practice, ZFS can store data encrypted, so ZFS not only provides a strong solution to offsite back-ups, it may also offer a strong solution to system update management (ie. poll for updates, clone current root fs and receive differential update against the clone, and reboot into the clone with automatic recovery back to the previous root on failure).

The purpose of this script is to provide a test-bed to determine whether the practical benefits of ZFS as a root filesystem are justifiable and whether the performance is adequate (especially around ram requirements).

## What is the Point (Long Answer)
This may be a bit of rambling, but we want to explain how we got here so we can get feedback from others and why we think this combination of technologies may have some unique benefits.

We like ZFS, specifically because it's ability to snapshot and clone datasets, as well as it's ability to send/receive snapshot differentials. Recently OpenZFS introduced encryption, leading to the possibility of encrypted offsite backups on untrusted servers (our real end-goal). Probably not completely untrusted servers, but perhaps "trusted backup archives" (trusted servers that store data they aren't able to access).

We do a lot of virtualization and containerization and like the CoreOS philosophy of a slim, standardized, and generally locked-down "VPS hypervisor" (technically it is not a hypervisor, but the idea is the same that the only thing it does is manage containers). But we were frustrated that CoreOS didn't necessarily have the things we wanted (such as sshfs) baked-in, and some things (such as ZFS) had overly complicated processes to get working (see https://github.com/varasys/corezfs for example).

Over the past couple years we have been evaluating different tools to achive these goals including various combinations of Debian, Ubuntu, ArchLinux, Alpine Linux, CoreOS, Docker, rkt, qemu, virtualbox, zfs, btrfs, overlayfs, kubernetes, etcd, and terraform (mostly on openstack or DigitalOcean VPS, but we are strictly VPS agnostic).

We found that the `pacman` package manager and ArchLinux repositories struck a perfect balance for what we wanted. The features we like are:

- official packages are distributed as binaries, and there is a `makepkg` utility to build from unofficial repositories (compare this with Gentoo `ebuild` which is more focused on compiling the system).

- official packages have minimal configuration, which is not good for recreational users (where .deb and .rpm are easier), but good for developers (like us) because it makes the upstream development document more relevant (ie. because you don't have to consider as many customizations from the package manager)

- it uses a "web of trust" trust model, which by default, includes five master keys pre-selected/installed by ArchLinux of five key developers which are used to sign the key of each trusted package developer (which are used to ultimately sign each package). If a package developer's key is signed by at least three of the five master keys it is considered trusted. This is just the default (and what most people use), but the point is that it is very customizable at a fine grained level so your production servers can use a trust model that only installs packages signed by your own master key and reject all other keys (ie. you would distribute packages from your own repository which first validates the source signature and then re-signs it with your master key).

The last point above deserves special explanation. We believe that the days of centralized CAs (ie. trust all of the root CAs installed by default) are numbered. Very soon it will not be possible for everyone to agree on common sets of trusted CAs. Several technologies are already anticipating this (such as DNSSEC) and being trialed and implemented with varying success. Although the pgp "web of trust" model has been around for along time, it is still not generally well known or well understood, but is very likely one of the fundamental technologies for the "post centralized CA" era.

We are also critically concerned (yes, we worry alot) about the overall robustness of the internet, and fully expect a fractured internet (ie. internets - plural) in the future where "identity portfolios" are required and are respected differently depending on the current "internet context". Again, we don't pretend to understand exactly how, but we feel that some sort of "web-of-trust" model will play a central role in this.

As we learned more about ArchLinux we liked the fact that ArchLinux has a very robust trust model (ie. trusting which packages can be installed) based on the "web-of-trust" model which is generalized to the point that it can be used completely independent of ArchLinux. We like ArchLinux so that is what we use; but we feel comfortable that if that ever changes or ArchLinux is compromised somehow we can just adjust the trust model configuration as needed.

So we learned how to use the `pacstrap` and `pacman` scripts from ArchLinux to bootstrap and ArchLinux machine and install OpenZFS.

We really liked the result, and the productivity boost! In our case, the productivity boost was because no matter what was happening or what container we were in we could instantly get any tool we wanted with pacman (on both the host and in the containers). Once we have things the way we want we can lock things down (ArchLinux is particularly good at uninstalling certain things) and use read-only containers. ZFS provides a productivity boost because you can snapshop, then clone a running container and open the clone to debug/troubleshoot.

Once we got this working we started being curious if running with the root filesystem on ZFS would be useful in any way. Off the top of our head it seemed like it could be used similar to the flash A/B partition method (using zfs differentials), and potentially could provide a very robust recovery boot method (like take a snapshot at completion of every successful boot, and use that as the bootfs if the next boot fails).

We found out that installing ArchLinux on a zfs root filesystem is surprisingly well documented. But the documentation wasn't immediately useful to us because we didn't understand the overall ArchZFS methodology/philosophy or the linux boot method. But after a little learning curve, we are really impressed with the ArchLinux documentation.

Our current interpretation of the ArchLinux philosophy is "a community based, security focused, mid-level, transparent, competent, and occasionally frustrating". Meaning:

- community based - ArchLinux doesn't seem to have any central authority and is all-in on the web-of-trust model. What this means in a practical sense is that you use a development machine to sign packages to distribute to production machines which only trust your development key (ie. the production servers don't necessarily have to trust the ArchZFS keys)

- security focused - ArchLinux seems to have security baked into it's ethos, such as it's fundamental trust model, and providing the linux-hardened kernel package (not to say other distros aren't security focused). This is based on our anecdotal experience though and part of our motivation in distributing this script is so more people can test this premise.

- mid-level - In this case just means that the `pacstrap` package management system typically distributes compiled, but minimally configured packages. We consider distro's such as Gentoo (based on `ebuild`) to be low-level since they are more focused on being able to compile a system from scratch (at least that is our impression). We consider distro's like Debian (.deb) and RedHat (.rpm) to be high level since they try and distribute compiled packages but then also make intelligent decisions to compile them. Since we spend most of our time in containers we find that "high-level" package managers that do special things to accommodate a generally arbitrary system state (such as required on a personal computer) are more of a nuisance than benefit. Note that ArchLinux also includes the `makepkg` utility which uses "PKGBUILD" scripts (similar to `ebuild`) which is convenient for compiling source code

- transparent - Although we haven't actually read and digested everything available (we are still testing things out also), we are really impressed by the level of documentation and transparency available in the ArchLinux ecosystem and believe the "web-of-trust" model is the currently available best technology for deploying software (we guess that most distro's use this to some extent, but we gravitated towards ArchLinux because their documentation about key management, although not without a learning curve, appears that it is feasible (although we haven't tried yet) to create a minimalistic "repo trust gateway" allowing packages to be distributed internally from repositories which apply a "trust model mapping" (for instance "if it meets the ArchLinux default then sign it with our internal production key and development key, otherwise sign it with our development key only"). We believe this has some powerful potential when combined with the ArchLinux User Repository (AUR) which is a community repository of PKGBUILD scripts (including lots of bleeding edge stuff)

- competent - The ArchLinux community seems to include people from every imaginable pocket of expertise. This probably applies to all distro's but once we started to understand "how to read" the ArchLinux documentation (and there is a little learning curve) we started to realize how complete and technically accurate it is; when combined with the upstream documentation. ArchLinux documentation does not "stand-alone" (in our opinion) the same way that documentation for other distributions does such as ubuntu, etc. Because the ArchLinux philosophy minimizes "package manager customizations" much of the upstream documentation applies directly and ArchLinux does not try and duplicate it. This is only a frustration until you remember to check the upstream documentation (which most of us were not in the habit of), and then we loved it. So now days we use high-level package managers alot to test things out, but when we find what we like we check-out how the high level package manager did it's configuration, but only as a reference (and not a limitation).

- occasionally frustrating - As much as we like `pacman` it is designed to do some pretty complex things such as upgrading existing systems. This is not useful to us because we don't upgrade systems; we re-containerize them (and hope to eventually use this rootfs on zfs to do A/B flash type system upgrades). Unfortunately though (for us at least) only the latest version of each package is available in the official ArchLinux repositories. ArchLinux does provide a solution to this in the form of the Arch Linux Rollback Machine (ARM) public repository which stores official repositories snapshots, iso images and bootstrap tarballs across time. Ideally, `pacman` could be configured to refer to the ARM if a package is not in the current repository (and maybe it can, but we haven't seen it and don't have time to research it). We doubt that it ever will (we guess because it's underlying algorithms just don't check dependencies that way). It also has some fundamental design decisions which are frustrating to script (such as having to pipe "Y" and "n" confirmations to `pacman` to ignore a package marked for ignoring). Again, it seems that the `pacman` developers are aware of these issues and unlikely to address them, probably because the current design decisions make sense in the context of a user applying rolling upgrades to a system (which is `pacman`'s original purpose we think).

So we found the biggest learning curve was learning the priority of documentation to get any given package installed, configured and running is the archlinux wiki first which usually provide a good overview of the package and which manual configuration steps are needed. Then the upstream documentation (which we were not in the habit of always checking) usually provides a good understanding of the rest. Online question and answer sites are also helpful, but it is important to understand which parts of the question/response are specific to the actual subject matter (vs. which are specific to customizations the users package manager has made which don't apply to ArchLinux).

We used to use CoreOS for our servers and Docker to run containers. The CoreOS philosophy is to run everything in Docker containers so the basic system doesn't do much except run Docker containers. There are some tools (such as toolbox) to help with system maintenance, but some of the tools that we love (such as `tmux` and `screen`) are conspicuously missing and surprisingly hard make available, which is just unacceptable (for our use case).  We spent a lot of time and effort learning how to even got zfs on CoreOS in the first place (refer to https://github.com/varasys/corezfs). In the process of learning how to get zfs modules compiled for CoreOS we learned about the systemd `systemd-nspawn` program (a `docker` alternative built-in to systemd) which seemed to address many of our frustrations we had with Docker (our philosophy about Docker volumes was always the opposite of Docker's; we only thought volumes were useful for throwaway data and real data should be bind mounted where it can be seen and properly backed-up, and we were never able to successfully create useful self-contained Dockerfiles, and typically ended up finding hacks to undo something the upstream Dockerfile author did).

Eventually we abandoned Docker altogether and focused on `systemd-nspawn`, and are still very happy with it. Similar to `pacman`, `systemd-nsapwn` doesn't fit our use case exactly (we find the .nspawn file method of configuring containers a bit clumsy and try to avoid it, which leads to applying `sed` to config files, etc.); but we are impressed with systemd and its continuing development and have full confidence that it is still a work in progress and generally hopeful, at this point, that systemd will only improve it. Also, we have and obsession with avoiding vendor lock-in, but accept that you have to start somewhere and have embraced the philosophy that now that systemd is ubiquitous we may as well use it's full potential. bash 4+ is another "lock-in" that we have committed to meaning it is always our first resort, which takes some getting used to because .

Although suspicious at first, we have grown to really like systemd. Similar (in philosophy) to ArchLinux, it can be installed in a pretty limited way to just init the system, but then has loads of great extra stuff such as management of the system dns resolver system (systemd-reselved.service), network management (systemd-networkd.service), log control (systemd-journald.service), and is very good at container management (`machinectl`). Systemd seems to follow the philosophy that it wants to give you everything you want, but won't force you to use it (ie. you can use dhcpcd.service instead of systemd-networkd.service if you want).

Systemd includes `machinectl` which is a utility to inspect and manage running containers. `systemd-nspawn` by default searches for container images or root directories in the /var/lib/machines folder and `machinectl` has some built-in functionality to prepare this folder as a btrfs volume (so it can use clones and snapshots to clone containers). We don't have much experience with btrfs and are already focused on a zfs based infrastructure so we don't use the btrfs support built into `machinectl` (we expect someday systemd will provide a zfs option, or use zfs if the /var/lib/machines directory is in a zfs dataset). The reason we want our container directories on zfs is so we can use `zfs send` to distribute updates from development servers to production servers.

This script is the result of adopting the CoreOS updating strategy without the CoreOS limitations. It is envisioned that to create a production server image you would first use this script to create a "development image" to use to
set-up the server exactly as needed (with the benefit of bash-completion, manual documents, and other utilities not strictly needed in the production server). You can "boot" your test production server using systemd-nspawn. When you are happy with it use "zpool set bootfs=zroot/system/new_image zroot" (adjusting as necessary to match the actual dataset and zpool name), and then reboot. After rebooting you can delete the original boot dataset (resulting in a bare bones customized OS image). Updates to production images can then be distributed as ZFS differential snapshots (using zfs send/receive). The default configuration includes seperate datasets for /home /root /srv /var/lib/machines and /var/lib/postgresql (so these will be the same no matter which root filesystem you are booted into). Use the "create-zfs-datasets" hook function to override this default configuration (see below).

`systemd-nspawn` has some built-in functionality to use btrfs as a backing filesystem for containers (in the /var/lib/machines directory) which allows cloning and snapshotting, but we were never comfortable with btrfs the same way we are with zfs (but we also don't have much btrfs experience). But for the purposes of managing containers in an image created from this script, you can create a container directory with:

```
zfs create zroot/machines/new_container
```
snapshot a container directory with:

```
zfs snapshot zroot/machines/new_container@snap_name
```
or clone a container directory with:

```
zfs clone zroot/machines/new_container@snap_name zroot/machines/clone_container
```

## DEPENDENCIES
zfs must be installed on the server running this script along with the following utilities and their dependencies. The version values below show version on ArchLinux of each utility that this script was created/tested with, grouped by the ArchLinux package containing them.
    
- core/bash=4.4.019-1
	- bash (4.4.19)
- core/pacman=5.0.2-2
	- pacman (5.0.2-2)
	- pacman-key (5.0.2-2)
	- pacstrap (n/a)
- core/coreutils=8.29-1
	- truncate (n/a)
- core/util-linux=2.31.1-1
	- lsblk (2.31.1)
	- blkid (2.31.1)
	- losetup (2.31.1)
	- mkswap (2.31.1)
- extra/gptfdisk=1.0.3-1
	- gdisk (1.0.3)
- extra/parted=3.2-6
	- partprobe (3.2)
- core/dosfstools=4.1-1
	- mkfs.vfat (4.1)
- archzfs/zfs-utils-common=0.7.6-3
	- zpool (n/a)
	- zfs (n/a)
- extra/qemu-headless=2.11.1-2 (or extra/qemu)
	- qemu-img (2.11.1) (only needed to create vdi, qcow, or qcow2 images)

## TODO
The following functionality is planned, but not incorporated yet. Please
raise an issue at https://github.com/varasys/paczfs if you are interested
in contributing to further development:

- dm-veritas image validation (still considering whether this is applicable/worthwhile, doesn't look too hard on the surface)
- BIOS boot loading support (it should just be a couple lines of script, but I don't know what they are and don't need BIOS support myself, but happy to implement if someone explains what is required or submits a patch)
- system hi-jack support (to install on VPS that can't upload images by unmounting current root and overwriting boot device with image from this script)
- root filesystem encryption support (in theory encrypted root is supported, but complicates boot loading and probably has no strong use case with respect to the root filesystem) (note that all other non-root datasets can already be encrypted)
- allow virtual machines to boot from the same rootfs as the running system (or any of it's snapshots) (the issue is that VMs usually want images, unlike containers which are happy with directories, so how to allow the system to present itself to a VM as an image (may not be possible) maybe something device mapper could help with? or may be better handled with overlayfs)
- Create ArchLinux User Repository (AUR) for this script (eventually to become a pkg if found useful (should be simple I think))

Thanks for your time, and we hope you have a pleasant experience ;-)
