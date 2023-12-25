+++
title = "Try impermanence with NixOS on a VM"
description = ""
date = 2023-12-25
draft = true

[taxonomies]
categories = ["Tech"]
tags = ["nixos", "linux", "impermanence"]

[extra]
toc = true
+++

Ever since reading Graham Christensen's blog post, [Erase your
darlings](https://grahamc.com/blog/erase-your-darlings), I've been intrigued by
the idea of opt-in state persistence. The concept has become known as
impermanence[^1] [^2], but I like to think of it as:

>I say we take off and nuke the entire site from orbit. [It’s the only way to be sure.](https://www.youtube.com/watch?v=tyyoaBa7DaE)

<!-- more -->

Before my last reinstall I had actually forgotten what would be in the `~/`
directory after a fresh install. Turns out nothing. The gradual accumulation of
cruft is annoying, and clutter has a negative effect on my state of mind. It's
distracting.

Left over cruft can breaks things. For example, the other day I couldn't login
to the graphical environment. I had to switch to a TTY to find out why with
`journalctl`:

```
systemd-xdg-autostart-generator[2375]: Exec binary 'teams' does not exist: No such file or directory
systemd-xdg-autostart-generator[2375]: /home/will/.config/autostart/teams.desktop: not generating unit, error parsing Exec= line: No such file or directory
```

A `~/.config/autostart/teams.desktop` was left over after uninstalling teams.<cite>[^1]</cite>

# The plan

- Create a custom NixOS live ISO
- Fire up the live ISO in a VM
- assume UEFI system (no instructions for legacy BIOS)
- tmpfs as root
- Partition with optimal alignment
- EXT4 for persistence <cite>[^2]</cite>
- With and without swap <cite>[^3]</cite>

Getting stuck in the planning stage is easy to do because of so many
opportunities for decision paralysis.

I'm largely using <https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/> as a guide
to use `tmpfs` (i.e. RAM) as root. I find the simplicity of its setup appealing.
I use a UPS with my system, so I want to give this a try before trying the ZFS /
Btrfs snapshots approach.

# Creating a custom NixOS live ISO

It would be nice to be able copy / paste between the host and the VM. If that's
not something you care about, you can skip this section.

Fortunately, it's easy [to create a custom live
CD/ISO](https://nixos.wiki/wiki/Creating_a_NixOS_live_CD):

`iso.nix`:

```nix
{ modulesPath, pkgs, ... }:
{
  imports = [
    "${modulesPath}/installer/cd-dvd/installation-cd-graphical-plasma5.nix"
  ];

  # Enables copy / paste when running in a KVM with spice.
  services.spice-vdagentd.enable = true;

  # Use faster squashfs compression
  isoImage.squashfsCompression = "gzip -Xcompression-level 1";
}
```


{% note(header="Note") %}

Changing`squashfsCompression` is not required, but it speeds up the build
dramatically for a modest trade-off in space.

{% end %}

Then build the `iso.nix` with:

```sh
nix-build '<nixpkgs/nixos>' \
  -A config.system.build.isoImage \
  -I nixos-config=iso.nix
```

The resulting ISO will be in `./result/iso/`.

In `iso.nix` I uses the graphical installer because X11 is needed for spice to
work. Check out the [imported
file](https://github.com/NixOS/nixpkgs/blob/f450b7ca7057b98367fc13e172d10f7dc2334db7/nixos/modules/installer/cd-dvd/installation-cd-graphical-plasma5.nix#L1)
to get a sense of how the live CD is configured.

I also made a flake based example on
[GitHub](https://github.com/willbush/ex-nixos-live-iso) which, in addition to
the above, also brings in
[home-manager](https://github.com/nix-community/home-manager) to add some
nice-to-haves such as:

- ZSH configured with completion and auto suggestions

- [Alacritty](https://github.com/alacritty/alacritty) with [starship prompt](https://starship.rs/)

- [fzf](https://github.com/junegunn/fzf) with ZSH integration (Ctrl + R etc.)

- [neovim](https://neovim.io/) configured with my custom keybindings made for
  Colemak-DH. I'm actually an Emacs user (with evil-mode), and I ought to try
  work on making my config less coupled to my
  [system](https://github.com/willbush/system).

- And more system packages like `tree`, `ripgrep`, and `mkpasswd`.

# KVM management tool

I'm using [virt-manager](https://virt-manager.org/) as the front-end for
libvirt. If you're on NixOS see: <https://nixos.wiki/wiki/Virt-manager>. I also
experimented with <https://cockpit-project.org/>, but ran into problems.

These days most computers use UEFI instead of BIOS for the firmware. I'm going
to assume no one needs BIOS to simplify these instructions.

For me, virt-manager uses BIOS by default, to change this:

1. Open virt-manager **⇒** View **⇒** set `x86 Firmware` to UEFI **⇒** Close

If you don't want to change the default, you can change it at VM creation time
after selecting the disk size:

1. Check the `Customize configuration before install` **⇒** Finish.

2. In the Overview section change the Firmware from BIOS to UEFI **⇒** Apply

# Fire up the VM

Using the ISO from the section about custom ISOs, or [an official
one](https://nixos.org/download.html#nixos-iso), `sudo mv` it into
`/var/lib/libvirt/images/` which is the default directory it looks for images. I
always end up having permission issues otherwise.

1. Open virt-manager **⇒** Select File **⇒** New virtual machine **⇒** Forward.

2. Choose ISO **⇒** Forward (it should auto-detect that it's NixOS).

3. Choose Memory and CPU amount (I personally use 25% - 50% of what I have available) **⇒** Forward.

4. Choose available disk size **⇒** Forward.

5. Begin Installation

Once booted into the ISO the first thing I do is right click on the desktop and
`Configure Display Settings` to change to a higher resolution.

Also consider, View **⇒** Scale Display Always, from the virt-manager window
running the VM.

# Partitioning

For me `lsblk` shows a disk named `vda`. Below replace `vda` with your disk.

{% note(header="Note") %}

One can wipe all the file systems on a device using: `sudo wipefs -a /dev/vda`
which is useful to start over or prepare a device.

{% end %}

Open a parted REPL: `sudo parted /dev/vda`

## Finding optimal alignment

Parted is a tricky tool to use from the command line especially when it comes to
getting optimal alignment. <cite>[^4]</cite> <cite>[^5]</cite> <cite>[^6]</cite>

When using `mkpart` with IEC units (e.g. MiB) or exact sectors parted will not
search for optimal alignment.

What we need to find is the starting sector of the first partition.

A simple trick to do this with `parted` is to:

```sh
# Since precentages are used,
# parted will automatically find the optimal alignment.
mkpart primary 0% 100%
# Print out the results showing units in sectors
unit s print
# Print out the results showing units in MiB
unit MiB print
# Remove the temporary partition
rm 1
```

For example:

```sh
$ sudo parted /dev/vda
GNU Parted 3.6
Using /dev/vda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mkpart primary 0% 100%
(parted) unit s print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 83886080s
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start  End        Size       File system  Name     Flags
 1      2048s  83884031s  83881984s               primary

(parted) unit MiB print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 40960MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start    End       Size      File system  Name     Flags
 1      1.00MiB  40959MiB  40958MiB               primary
(parted) rm 1
```

So I will using `1MiB` (which is the same as `2048s`) as the starting sector for
the first partition.

## Create partitions

First lets look at the annotated commands:

```sh
# default units for `print` and `mkpart` commands
unit MiB
# initialize the partition table
mklabel gpt
# Make boot partition 512 MiB.
# Note that 1MiB is where my partition needs to start for optimal alignment.
mkpart ESP fat32 1 513
# mark the partition is bootable
set 1 boot on
# Make swap partition 8705 = 513 + 8192 (note 8192MiB = 8GiB)
mkpart swap linux-swap 513 8705
# Make root partition the rest of the drive
mkpart nix 8705 100%
# see the results
print
# Check alignment on each partition
align-check optimal 1
align-check optimal 2
align-check optimal 3
q
```

Here are the commands with repl output:

```sh
$ sudo parted /dev/vda
GNU Parted 3.6
Using /dev/vda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) unit MiB
(parted) mklabel gpt
(parted) mkpart ESP fat32 1 513
(parted) set 1 boot on
(parted) mkpart swap linux-swap 513 8705
(parted) mkpart nix 8705 100%
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 30720MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start    End       Size      File system     Name  Flags
 1      1.00MiB  513MiB    512MiB    fat32           ESP   boot, esp
 2      513MiB   8705MiB   8192MiB   linux-swap(v1)  swap  swap
 3      8705MiB  30719MiB  22014MiB                  nix

(parted) align-check optimal 1
1 aligned
(parted) align-check optimal 2
2 aligned
(parted) align-check optimal 3
3 aligned
(parted) q
```

If you don't want swap [^3], then just change these lines:

```diff
-mkpart swap linux-swap 513 8705
-mkpart nix 8705 100%
+mkpart nix 513 100%
```

The `lsblk` command should now show the partitions on the disk:

```sh
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  3.1G  1 loop /nix/.ro-store
sr0     11:0    1  3.2G  0 rom  /iso
vda    253:0    0   30G  0 disk
├─vda1 253:1    0  512M  0 part
├─vda2 253:2    0    8G  0 part
└─vda3 253:3    0 21.5G  0 part
```

# Format

```sh
$ sudo mkfs.fat -F 32 -n boot /dev/vda1
$ sudo mkswap -L swap /dev/vda2
$ sudo mkfs.ext4 -L nixos /dev/vda3
```

For no swap:

```diff
-sudo mkswap -L swap /dev/vda2
-sudo mkfs.ext4 -L nixos /dev/vda3
+sudo mkfs.ext4 -L nixos /dev/vda2
```

Verify the partitions and file systems using. Here's what I get with a 40GiB
virtual disk:

```sh
$ sudo parted /dev/vda -- unit MiB print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 40960MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start    End       Size      File system     Name  Flags
 1      1.00MiB  513MiB    512MiB    fat32           ESP   boot, esp
 2      513MiB   8705MiB   8192MiB   linux-swap(v1)  swap  swap
 3      8705MiB  40959MiB  32254MiB  ext4            nix
```

# Mount

Here we do one of the neat tricks, we mount tmpfs instead of a partition and then we mount the partitions we just created.

```sh
mount -t tmpfs none /mnt
mkdir -p /mnt/{boot,nix,etc/nixos,var/log}
mount /dev/vda1 /mnt/boot
mount /dev/vda2 /mnt/nix
mkdir -p /mnt/nix/persist/{etc/nixos,var/log}
mount -o bind /mnt/nix/persist/etc/nixos /mnt/etc/nixos
mount -o bind /mnt/nix/persist/var/log /mnt/var/log
```

```sh
# Mount your root file system
mount -t tmpfs none /mnt

# Create directories
mkdir -p /mnt/{boot,nix,etc/nixos,var/log}

# Mount /boot and /nix
mount $DISK-part1 /mnt/boot
mount $DISK-part2 /mnt/nix

# Create a directory for persistent directories
mkdir -p /mnt/nix/persist/{etc/nixos,var/log}

# Bind mount the persistent configuration / logs
mount -o bind /mnt/nix/persist/etc/nixos /mnt/etc/nixos
mount -o bind /mnt/nix/persist/var/log /mnt/var/log
```

# Installation

```
sudo swapon /dev/vda2
```

---

[^1] <https://nixos.wiki/wiki/Impermanence>

[^2] <https://github.com/nix-community/impermanence>

[^3] Why this breaks the graphical environment I still don't know. Perhaps I need to disable `xdg.autostart.enable` which defaults to true?

[^3] I spent way too much time considering if I want to switch away from EXT4.
After considering features and performance of file systems such as Btrfs, ZFS,
f2fs, XFS, bcachefs, I decided to stick with EXT4 for my workstation. Though I
plan to revisit this topic again in the future.

[^4] https://wiki.archlinux.org/title/Parted#Alignment

[^5] https://blog.hqcodeshop.fi/archives/273-GNU-Parted-Solving-the-dreaded-The-resulting-partition-is-not-properly-aligned-for-best-performance.html

[^6] https://unix.stackexchange.com/questions/38164/create-partition-aligned-using-parted/401118#401118

[^7] This is always a controversial topic. See [an argument in favor of swap](https://chrisdown.name/2018/01/02/in-defence-of-swap.html).
