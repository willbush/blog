+++
title = "Try NixOS with impermanence on a VM"
description = ""
date = 2023-12-24
draft = true

[taxonomies]
categories = ["Tech"]
tags = ["nixos", "linux", "impermanence"]

[extra]
toc = true
+++

Ever since reading Graham Christensen's blog post, [Erase your
darlings](https://grahamc.com/blog/erase-your-darlings), I've been intrigued by
the idea of opt-in state persistence.

This quote from the movie Aliens is what it reminds me of:

>I say we take off and nuke the entire site from orbit. It’s the only way to be sure.

This post documents my journey of starting fresh with
[impermanence](https://nixos.wiki/wiki/Impermanence) in NixOS.

<!-- more -->

Usually around this time of year I do my [backup
ritual](https://willbush.dev/blog/synology-nas-backup/), and in the process I
reinstall NixOS just for that "new computer smell."

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
- Cover how to try this out in a VM
- assume UEFI system (no instructions for BIOS)
- tmpfs as root
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
- And more system packages

# Fire up the VM

I'm using [virt-manager](https://virt-manager.org/) as the front-end for
libvirt. If you're on NixOS see: <https://nixos.wiki/wiki/Virt-manager>.

Using the custom ISO from the previous section or [an official
one](https://nixos.org/download.html#nixos-iso), `sudo mv` it into
`/var/lib/libvirt/images/` which is the default directory it looks for images. I
always end up having permission issues otherwise.

1. Open virt-manager **⇒** Select File **⇒** New virtual machine **>** Forward.

2. Choose ISO **⇒** Forward (it should auto-detect that it's NixOS).

3. Choose Memory and CPU amount (I personally use 25% - 50% of what I have available) **⇒** Forward.

4. Choose available disk size **⇒** Forward.

5. Check the `Customize configuration before install` **⇒** Finish.

6. In the Overview section change the Firmware from BIOS to UEFI **⇒** Apply

7. Begin Installation

Once booted into the ISO the first thing I do is right click on the desktop and
`Configure Display Settings` to change to a higher resolution.

Also consider, View **⇒** Scale Display Always, from the virt-manager window
running the VM.

# Partitioning and formatting

Almost all commands in this section need to be run as root, so `sudo su` to switch to root.

For me `lsblk` shows a disk named `vda`. Below replace `vda` with your disk.

{% note(header="Note") %}

One can wipe all the file systems on a device using: `sudo wipefs -a /dev/vda`
which is useful to start over or prepare a device.

{% end %}

Open a parted REPL: `parted`

```
select /dev/vda
mklabel gpt
mkpart ESP fat32 1MiB 512MiB
set 1 boot on
mkpart nix 512MiB 100%
q
```

The `lsblk` command should now show the partitions on the disk.

```sh
mkfs.fat -F 32 -n boot /dev/vda1
mkfs.ext4 -L nixos /dev/vda2
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

---

[^1] Why this breaks the graphical environment I still don't know. Perhaps I need to disable `xdg.autostart.enable` which defaults to true?

[^2] I spent way too much time considering if I want to switch away from EXT4.
After considering features and performance of file systems such as Btrfs, ZFS,
f2fs, XFS, bcachefs, I decided to stick with EXT4 for my workstation. Though I
plan to revisit this topic again in the future.

[^3] This is always a controversial topic. See [an argument in favor of swap](https://chrisdown.name/2018/01/02/in-defence-of-swap.html).

<https://github.com/nix-community/impermanence>

<https://nixos.wiki/wiki/Impermanence>
