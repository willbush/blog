+++
title = "Try impermanence with NixOS on a VM"
date = 2023-12-25
draft = true

[taxonomies]
categories = ["Tech"]
tags = ["nixos", "linux", "impermanence"]

[extra]
toc = true
comment = true
copy = true
math = false
mermaid = false
+++

Ever since reading Graham Christensen's blog post, [Erase your
darlings](https://grahamc.com/blog/erase-your-darlings), I've been intrigued by
the idea of opt-in state persistence. The concept has become known as
impermanence[^1] [^2], but I like to think of it as:

>I say we take off and nuke the entire site from orbit. [It’s the only way to be sure.](https://www.youtube.com/watch?v=tyyoaBa7DaE)

<!-- more -->

So what is impermanence? To quote the [nix-community impermanence
readme](https://github.com/nix-community/impermanence):

>Lets you choose what files and directories you want to keep between reboots - the rest are thrown away.
>
>Why would you want this?
>
>- It keeps your system clean by default.
>- It forces you to declare settings you want to keep.
>- It lets you experiment with new software without cluttering up your system.

Before my last reformat, I had actually forgotten what would be in the `~/`
directory after freshly installing NixOS with no desktop environment. Turns out
nothing. The gradual accumulation of cruft is annoying, and clutter has a
negative effect on my state of mind. It's distracting.

Leftover cruft can break things. For example, the other day I couldn't log in to
the graphical environment. I had to switch to a TTY to find out why using
`journalctl`:

```
systemd-xdg-autostart-generator[2375]: Exec binary 'teams' does not exist: No such file or directory
systemd-xdg-autostart-generator[2375]: /home/will/.config/autostart/teams.desktop: not generating unit, error parsing Exec= line: No such file or directory
```

A `~/.config/autostart/teams.desktop` was left over after uninstalling
teams[^3]. Also, crap like this trying to bootstrap autostart on
its own is exactly the sort of thing I want to nuke from orbit.

# The plan

- Create a custom NixOS live ISO
- Fire up the live ISO in a VM
- assume UEFI system (no instructions for legacy BIOS)
- tmpfs as root
- Partition with optimal alignment
- EXT4 for persistence [^4]
- With or without swap [^5]

This post is going to be a walk-through of how to try out impermanence with NixOS in a VM.

I personally try this sort of thing in a VM first. I did so when migrating to
NixOS. And again, when I rewrote my config for nix
[flakes](https://nixos.wiki/wiki/Flakes.). This time, I want to document it
better, and hopefully make it easier for others to try it out too.

I'm largely using <https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/> as a guide
to use `tmpfs` (i.e. RAM) as root. I find the simplicity of its setup appealing.
However, power outages create a risk of data loss. I use a UPS with my system,
and want to give this a try before trying the ZFS / Btrfs snapshot approach.

# Creating a custom NixOS live ISO

It's nice to be able to copy and paste between the host and the VM. If that's
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

In `iso.nix` I use the graphical installer because X11 is needed for spice to
work. Check out the [imported
file](https://github.com/NixOS/nixpkgs/blob/f450b7ca7057b98367fc13e172d10f7dc2334db7/nixos/modules/installer/cd-dvd/installation-cd-graphical-plasma5.nix#L1)
to get a sense of how the live CD is configured.

I also made a flake based [example on
GitHub](https://github.com/willbush/ex-nixos-live-iso) which, in addition to the
above, also brings in
[home-manager](https://github.com/nix-community/home-manager) to add some
nice-to-haves such as:

- ZSH configured with completion and auto suggestions

- [Alacritty](https://github.com/alacritty/alacritty) with [starship prompt](https://starship.rs/)

- [fzf](https://github.com/junegunn/fzf) with ZSH integration (Ctrl + R etc.)

- [neovim](https://neovim.io/) with a minimal config for my custom keybindings
  made for [Colemak-DH](https://colemakmods.github.io/mod-dh/). I'm actually an
  Emacs user (with evil-mode), but that's overkill for this situation.

- xclip for clipboard support in neovim, and other packages I could have
  installed on the fly with nix-env.

Feel free to fork it and/or clone and modify it to your liking.

# KVM management tool

I'm using [virt-manager](https://virt-manager.org/) as the front-end for
libvirt. If you're on NixOS see: <https://nixos.wiki/wiki/Virt-manager>. I also
experimented with <https://cockpit-project.org/>, but ran into problems.

These days most computers use UEFI instead of BIOS for the firmware. I'm going
to assume no one needs BIOS to simplify these instructions.

For me, virt-manager uses BIOS by default, to change this:

1. Open virt-manager **⇒** View **⇒** set x86 Firmware to UEFI **⇒** Close

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
which is useful to start over after a mistake or to prepare a device that
already has file systems.

{% end %}

Open a parted REPL: `sudo parted /dev/vda`

## Finding optimal alignment

Parted is a tricky tool to use from the command line especially when it comes to
getting optimal alignment. [^6] [^7] [^8]

When using `mkpart` with IEC units (e.g. MiB) or exact sectors parted will not
search for optimal alignment. The NixOS manual used to use MiB / GiB for parted,
but [it got changed](https://github.com/NixOS/nixpkgs/issues/276000) to MB / GB
to help avoid alignment issues (parted searches a radius around the given
position). I prefer to have my partitions in nice clean IEC unit sizes because
that's what most tools default to (e.g. gparted, cfdisk, lsblk, `df -h` etc.).

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

Here are the same commands with repl output:

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

If you don't want swap [^5], then just change these lines:

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

I'm going to base the following on the excellent
<https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/> guide.

```sh
# Mount your root file system as tmpfs
sudo mount -t tmpfs none /mnt

# Create mount directories
sudo mkdir -p /mnt/{boot,nix,etc/nixos,var/log}

# Mount /boot and /nix
sudo mount /dev/vda1 /mnt/boot
sudo mount /dev/vda3 /mnt/nix

# Create persistent directories
sudo mkdir -p /mnt/nix/persist/{etc/nixos,var/log}

# Bind mount the persistent configuration / logs
sudo mount -o bind /mnt/nix/persist/etc/nixos /mnt/etc/nixos
sudo mount -o bind /mnt/nix/persist/var/log /mnt/var/log
```

# Configure

If you're using swap, enable it now. Otherwise, it won't get automatically
configured in your `hardware-configuration.nix`:

```sh
$ sudo swapon /dev/vda2
```

Generate the initial `configuration.nix` and `hardware-configuration.nix`:

```sh
$ sudo nixos-generate-config --root /mnt
```

```sh
$ sudo -E nvim /mnt/etc/nixos/hardware-configuration.nix
```

Per Elis's blog post, we need to set some options on the tmpfs root in the
`hardware-configuration.nix`, so open it for editing (e.g. `sudo -E nvim
/mnt/etc/nixos/hardware-configuration.nix`):

>The most important bit is the `mode`, otherwise certain software (such as
>openssh) won't be happy with the permissions of the file system.
>
>The `size` is something you can adjust depending on how much garbage you are
>willing to store in ram until you run out of space on your root. 2G is usually
>big enough for most of my systems.

```nix
{
  #...
  fileSystems."/" =
    { device = "none";
      fsType = "tmpfs";
      options = [ "defaults" "size=2G" "mode=755" ];
    };
  #...
}
```

Next edit `configuration.nix` to your liking, but the password should not be mutable.

Generate a hashed password. The following is an example:

```sh
$ nix-shell --run 'mkpasswd -m SHA-512 -s' -p mkpasswd
Password: your password
<hash output>
```

Edit `configuration.nix` to disable `mutableUsers` and paste in your password hash:

```nix
{
  #...

  users.mutableUsers = false;
  users.users.root.initialHashedPassword = "<hash output>";
  #...
}
```

{% important(header="Important") %}

Be sure to use `initialHashedPassword` instead of `hashedPassword` because the
latter doesn't work for this setup.

{% end %}

# Install

```sh
sudo nixos-install --no-root-passwd
```

---

[^1]: <https://nixos.wiki/wiki/Impermanence>

[^2]: <https://github.com/nix-community/impermanence>

[^3]: Why this breaks the graphical environment I still don't know. Perhaps I
    need to disable `xdg.autostart.enable` which defaults to true?

[^4]: I spent way too much time considering whether I wanted to switch away from
EXT4. After considering the features and performance of file systems such as:

- Btrfs
- ZFS
- f2fs
- XFS
- bcachefs

I decided to stick with EXT4 for my workstation. Though I plan to revisit this
topic again in the future.

[^5]: This is always a controversial topic. And I can't blame either position
because I have yet to find any argument with experiments to back up claims.
Here's [an argument in favor of
swap](https://chrisdown.name/2018/01/02/in-defence-of-swap.html) which I keep
running across.

[^6]: <https://wiki.archlinux.org/title/Parted#Alignment>

[^7]: <https://blog.hqcodeshop.fi/archives/273-GNU-Parted-Solving-the-dreaded-The-resulting-partition-is-not-properly-aligned-for-best-performance.html>

[^8]: <https://unix.stackexchange.com/questions/38164/create-partition-aligned-using-parted/401118#401118>

