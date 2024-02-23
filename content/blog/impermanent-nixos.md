+++
title = "Impermanent NixOS: on a VM + tmpfs root + flakes + LUKS"
date = 2024-01-29
draft = false

[taxonomies]
categories = ["Tech"]
tags = ["nixos", "linux", "impermanence"]

[extra]
toc = true
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

Check out the [video I made](https://www.youtube.com/watch?v=Nts-z1oqo6M)
introducing this blog post. It might be a good starting point to add some
context of what you're getting yourself into.

So what is impermanence? To quote the [nix-community impermanence
readme](https://github.com/nix-community/impermanence):

>Lets you choose what files and directories you want to keep between reboots - the rest are thrown away.
>
>Why would you want this?
>
>- It keeps your system clean by default.
>- It forces you to declare settings you want to keep.
>- It lets you experiment with new software without cluttering up your system.

The other day I couldn't log in to the graphical environment on my system. I had
to switch to a TTY to find out why using `journalctl`:

```
systemd-xdg-autostart-generator[2375]: Exec binary 'teams' does not exist: No such file or directory
systemd-xdg-autostart-generator[2375]: /home/will/.config/autostart/teams.desktop: not generating unit, error parsing Exec= line: No such file or directory
```

A `~/.config/autostart/teams.desktop` was left over after uninstalling
teams[^3]. Crap like this trying to bootstrap autostart on its own is exactly
the sort of thing I want to nuke from orbit. So I'm finally going to give
impermanence a try.

# The plan

- Create a custom NixOS live ISO
- Fire up the live ISO in a VM
- assume UEFI system (no instructions for legacy BIOS)
- tmpfs as root
- Partition with optimal alignment
- Optional LUKS encryption on root
- EXT4 for persistence [^4]
- With swap [^5] optionally encrypted with random key
- An opinionated install using nix [flakes](https://nix.dev/concepts/flakes.html) [^6]

This post is going to be a walk-through of how to try out impermanence with
NixOS in a VM. I have also tested this on a Framework laptop with a NVMe drive.
I have run through the steps many times in a VM to make sure they work and are
easy to copy, paste, and run.

{% note(header="Important") %}

**I encourage reviewing the code before running.**

{% end %}

I imagine the audience for this post is people who are already
familiar with NixOS, but I'll try to keep in mind those crazy enough to try
NixOS, flakes, and impermanence for the first time.

I personally try this sort of thing in a VM first. For example, when migrating
to NixOS. And again, when I rewrote my config for nix flakes. This time, as I'm
migrating to impermanence, I want to document the process better. Hopefully,
this will make it easier for others to try it out too.

I'm largely using <https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/> as a guide
to use `tmpfs` (i.e. RAM) as root. I find the simplicity of its setup appealing.
However, power outages create a risk of data loss. I use a UPS with my system,
and want to give this a try before trying the ZFS / Btrfs snapshot approach.

# Creating a custom NixOS live ISO

It's nice to be able to copy and paste between the host and the VM, and the
following code blocks are written to make them easy to paste and run.

Fortunately, it's easy [to create a custom live CD/ISO](https://nixos.wiki/wiki/Creating_a_NixOS_live_CD):

If that's something you rather not mess with, then skip this section.

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

{% note(header="Note") %}

If you don't have Nix installed, I recommend using the Determinate Systems [Nix
installer](https://zero-to-nix.com/concepts/nix-installer) (reasons given on the
website):

{% end %}

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

Feel free to fork it and modify it to your liking.

# KVM management tool

I'm using [virt-manager](https://virt-manager.org/) as the front-end for
libvirt. If you're on NixOS, see: <https://nixos.wiki/wiki/Virt-manager> for
install instructions. I also experimented with <https://cockpit-project.org/>,
but ran into problems.

These days, most computers use UEFI instead of BIOS for the firmware. I'm going
to assume no one needs BIOS to simplify these instructions.

For me, virt-manager uses BIOS by default, to change this:

1. Open virt-manager **⇒** View **⇒** set x86 Firmware to UEFI **⇒** Close

If you don't want to change the default, you can change it at VM creation time
after selecting the disk size:

1. Check the `Customize configuration before install` **⇒** Finish.

2. In the Overview section, change the Firmware from BIOS to UEFI **⇒** Apply

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

Once booted into the ISO, the first thing I do is right-click on the desktop and
`Configure Display Settings` to change to a higher resolution.

# Partitioning

For me, `lsblk` shows a disk named `vda`. Replace `vda` with your disk below.

Set up a variable for the disk which will be used in the following commands:

```sh
export DISK=/dev/vda
```

{% note(header="Note") %}

I'm going to use `1MiB` (sector 2048) as the starting sector for optimal
alignment, and that will probably work for you too. However, if you run into
alignment issues, then see the "Finding optimal alignment" section at the end of
the blog post.

{% end %}


```sh
sudo parted $DISK --script \
  unit MiB \
  mklabel gpt \
  mkpart ESP fat32 1 513 \
  set 1 boot on \
  mkpart swap linux-swap 513 8705 \
  mkpart nix 8705 100% \
  print
```

Here are the annotated `parted` commands from above:

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
```

The `lsblk` command should now show the partitions on the disk:

```sh
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  3.1G  1 loop /nix/.ro-store
sr0     11:0    1  3.2G  0 rom  /iso
vda    253:0    0   40G  0 disk
├─vda1 253:1    0  512M  0 part
├─vda2 253:2    0    8G  0 part
└─vda3 253:3    0 31.5G  0 part
```

Check alignment:

```sh
for i in {1..3}; do sudo parted $DISK -- align-check optimal $i; done
```

# Format

The following will set variables `PART1`, `PART2`, and `PART3` to the partition
names. I realized when trying this out on a laptop with a NVMe drive that simply
using `${DISK}1` wouldn't work when the disk ends with a number such as
`nvme0n1`, the partitions end up looking like `nvme0n1p1` etc.

```sh
for i in {1..3};\
  do export "PART$i"=$(lsblk -lp | grep part | grep ${DISK} | awk -v line=$i 'NR==line{print $1}');\
done;\
echo $PART1; echo $PART2; echo $PART3
```

The output should look similar to:

```
/dev/vda1
/dev/vda2
/dev/vda3
```

**Optionally**, LUKS encrypt the root partition:

```sh
sudo cryptsetup luksFormat $PART3
```

Open the encrypted partition and change the variable to point to the decrypted
partition:

```sh
sudo cryptsetup luksOpen $PART3 crypted && \
  export PART3=/dev/mapper/crypted
```

Now format the partitions:

```sh
sudo mkfs.fat -F 32 -n boot ${PART1} && \
  sudo mkswap -L swap ${PART2} && \
  sudo mkfs.ext4 -L nixos ${PART3}
```

I always ignore the warning:

>lowercase labels might not work properly on some systems

Visually verify the partitions and file systems using:

```sh
sudo parted $DISK -- unit MiB print
```

Here's what I get with a 40GiB virtual disk:

```
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
cat << 'EOF' > ~/mount.sh
#!/usr/bin/env bash

set -e

# Mount your root file system as tmpfs
mount -v -t tmpfs none /mnt

# Create mount directories
mkdir -v -p /mnt/{boot,nix,etc/nixos,var/log}

# Mount /boot and /nix
mount -v $PART1 /mnt/boot -o umask=0077
mount -v $PART3 /mnt/nix

# Create persistent directories
mkdir -v -p /mnt/nix/persist/{etc/nixos,var/log}

# Bind mount the persistent configuration / logs
mount -v -o bind /mnt/nix/persist/etc/nixos /mnt/etc/nixos
mount -v -o bind /mnt/nix/persist/var/log /mnt/var/log

# Make config directory temporarily easier to work with
chmod -v 777 /mnt/etc/nixos

EOF
```

```sh
chmod u+x ~/mount.sh && sudo -E ~/mount.sh
```

# Configure hardware

Enable swap now. Otherwise, it won't get automatically configured in your
`hardware-configuration.nix`:

```sh
sudo swapon $PART2
```

Generate the initial `configuration.nix` and `hardware-configuration.nix`:

```sh
nixos-generate-config --root /mnt && cd /mnt/etc/nixos
```

If you see the following, just ignore it:

>ERROR: Not a Btrfs filesystem: Invalid argument

Per Elis's blog post, we need to set some options on the tmpfs root in the
`hardware-configuration.nix`:

>The most important bit is the `mode`, otherwise certain software (such as
>openssh) won't be happy with the permissions of the file system.
>
>The `size` is something you can adjust depending on how much garbage you are
>willing to store in ram until you run out of space on your root...

```diff
{
  #...
  fileSystems."/" =
    { device = "none";
      fsType = "tmpfs";
+     options = [ "defaults" "size=25%" "mode=755" ];
    };
  #...
}
```

The following will add the options to the tmpfs root and format the two nix
files using `nixpkgs-fmt`:

```sh
sed -i '/fsType = "tmpfs";/a options = [ "defaults" "size=25%" "mode=755" ];' \
  ./hardware-configuration.nix && \
  nix-shell -p nixpkgs-fmt --run 'nixpkgs-fmt .'
```

## Increase security of the boot mount

You may have noticed when I mounted `boot` I used `umask=0077`. This was to avoid the following warning:

```
⚠️ Mount point '/boot' which backs the random seed file is world accessible, which is a security hole! ⚠️
⚠️ Random seed file '/boot/loader/.#bootctlrandom-seedc878009c19a876dc' is world accessible, which is a security hole! ⚠️
```

There's a [nixpkgs issue](https://github.com/NixOS/nixpkgs/issues/279362) about
it. The generated `hardware-configuration.nix` does not currently pick up these
permission options. So let's fix that again using `sed`.

```sh
sed -i '/fsType = "vfat"/a options = [ "umask=0077" ];' \
  ./hardware-configuration.nix && \
  nix-shell -p nixpkgs-fmt --run 'nixpkgs-fmt .'
```

## Optional encrypted swap

If you're using LUKS on the root partition, then you might want to [encrypt
swap](https://nixos.wiki/wiki/Swap). However, the `by-uuid` generated for the
`swapDevice` in the `hardware-configuration` needs to change to the
`by-partuuid` because when using the `randomEncryption.enable = true` the
`by-uuid` changes every boot.

To keep with the theme of the post, I threw together a script to do the work for
you:

```sh
cat << 'EOF' > ~/encrypt-swap.sh
#!/usr/bin/env bash

set -e

if [[ -z $PART2 ]]; then
  echo "PART2 is undefined or empty"
  exit 1
fi

hwConfig=/mnt/etc/nixos/hardware-configuration.nix
backupHwConfig=/mnt/etc/nixos/hardware-configuration.backup.nix

main() {
  swapPart=$(echo $PART2 | awk -F'/' '{print $NF}')
  swapDiskUUID=$(ls -l /dev/disk/by-uuid | grep $swapPart | awk '{print $9}')
  swapPartUUID=$(ls -l /dev/disk/by-partuuid | grep $swapPart | awk '{print $9}')

  echo "swapDiskUUID: $swapDiskUUID"
  echo "swapPartUUID: $swapPartUUID"

  sed -i "s|by-uuid/$swapDiskUUID|by-partuuid/$swapPartUUID|g" $hwConfig
  sed -i "/$swapPartUUID/s/\";/\";\n/" $hwConfig
  sed -i "/$swapPartUUID\"/ a\\randomEncryption.enable = true;" $hwConfig

  nix-shell -p nixpkgs-fmt --run "nixpkgs-fmt $hwConfig"
}

cp $hwConfig $backupHwConfig
trap 'cp $backupHwConfig $hwConfig' ERR
main
EOF
```

```sh
chmod u+x ~/encrypt-swap.sh && ~/encrypt-swap.sh
```

# Configure with flakes

If you're not familiar with nix flakes, I recommend reading at least the first
link below:

- <https://nix.dev/concepts/flakes.html>
- <https://www.tweag.io/blog/2020-05-25-flakes/>
- <https://nixos.wiki/wiki/Flakes>

We're getting to the [rest of the
owl](https://knowyourmeme.com/memes/how-to-draw-an-owl) section that seems
inevitable with configuring NixOS. To help out, I made an example [starter
config](https://github.com/willbush/ex-nixos-starter-config) that I configured
using
[Misterio77/nix-starter-configs](https://github.com/Misterio77/nix-starter-configs)
minimal template. So if you're starting out with flakes, then I recommend
checking out both. I'll talk about this more in the last section of the blog
post.

Here's how to use my starter config:

```sh
git clone https://github.com/willbush/ex-nixos-starter-config.git && \
  mv hardware-configuration.nix ./ex-nixos-starter-config && \
  cd ex-nixos-starter-config && \
  git add .
```

{% note(header="Note") %}

New files in a git repo must be staged at least for flake related commands to be
aware of them. This is why `git add .` is needed so that the
`hardware-configuration.nix` is staged.

Otherwise, an error like the following will happen:

error: getting status of '/mnt/nix/store/21zpkqcn55a73x9y8yy4lrrd7ja3mjvc-source/nixos/hardware-configuration.nix': No such file or directory

{% end %}

# Install

Before installing, I recommend looking over the generated
`/mnt/etc/nixos/configuration.nix`. It's not going to be used in the
installation, so copy any settings you want into the
`/mnt/etc/nixos/ex-nixos-starter-config/configuration.nix` file.

If you like, rename [blitzar](https://en.wikipedia.org/wiki/Blitzar) to whatever
hostname you like. I usually name my hosts after astronomical bodies or concepts
for fun.

From the `/mnt/etc/nixos/ex-nixos-starter-config` directory run:

```sh
NIX_CONFIG="experimental-features = nix-command flakes" \
  sudo nixos-install --flake .#blitzar --no-root-passwd
```

Change `/mnt/etc/nixos` permissions back and reboot:

```sh
sudo chmod -v 755 /mnt/etc/nixos && \
reboot
```

{% note(header="Note") %}

Since we're using Nix flakes, there's no requirement for the repo to be under
`/etc/nixos`. Feel free to change it after rebooting, but make sure you persist
the directory you decide to store it!

{% end %}

If you're using my example starter config, then see its
[readme](https://github.com/willbush/ex-nixos-starter-config) for credentials
and how to change them.

# Rest of the owl

One [important
difference](https://github.com/willbush/ex-nixos-starter-config/commit/7009970b2b0b1859ce108d3364ccd94387397b41)
I made when configuring from the minimal
[Misterio77/nix-starter-configs](https://github.com/Misterio77/nix-starter-configs)
template is I switched [home
manager](https://github.com/nix-community/home-manager) to a
[module](https://nix-community.github.io/home-manager/index.xhtml#sec-flakes-nixos-module)
within a NixOS system configuration.

This means all the `home.nix` configuration is applied when `nixos-install` is
run (above) or in the more normal case when using `nixos-rebuild`. I rather not
have to use the `home-manager` standalone CLI tool. I suppose one might have a
use-case where they want to configure their home independently of their system.
I'm not sure what the best approach is to avoid having to run the tool on every
boot.

## Finding what to perist

Hi, it's me from the future. I've been using this setup for a couple of weeks.
One thing that's helpful is to be able to find differences between the
`/nix/persist` and `/` directories. I use `rsync` with `--dry-run` to do this:

```sh
sudo rsync -amvxx \
  --dry-run \
  --no-links \
  --exclude '/tmp/*' \
  --exclude '/root/*' \
  / /nix/persist/ \
  | rg -v '^skipping|/$'
```

Here is some documentation on the options used:

```
     --no-OPTION             turn off an implied OPTION (e.g. --no-D)
 -a, --archive               archive mode; equals -rlptgoD (no -H,-A,-X)
 -c, --checksum              skip based on checksum, not mod-time & size
 -m, --prune-empty-dirs      prune empty directory chains from file-list
 -n, --dry-run               perform a trial run with no changes made
 -v, --verbose               increase verbosity
 -x, --one-file-system       don't cross filesystem boundaries

    If this option is repeated, rsync omits all mount-point directories from the
    copy. Otherwise, it includes an empty directory at each mount-point it
    encounters (using the attributes of the mounted directory because those of
    the underlying mount-point directory are inaccessible).
```

However, note I'm not using `-c, --checksum` because it's slower and seems to be
overkill.

I exclude `/tmp` and `/root` because I don't want to persist those directories.

I use `rg` ([ripgrep](https://github.com/BurntSushi/ripgrep)) to filter out results.
Such as lines starting with `skipping`:
```
skipping non-regular file "home/will/.local/state/home-manager/gcroots/current-home"
skipping non-regular file "home/will/.local/state/nix/profiles/home-manager"
...
```

or ending with `/`:

```
home/will/.config/hypr/
home/will/.config/mako/
home/will/.config/nvim/
home/will/.config/swaylock/
home/will/.config/systemd/
...
```

You can also reverse the source / destination to find orphaned files in the
`/nix/persist` directory.

## Documentation

Of course, the real "rest of the owl" is actually configuring your system, and
there's no way around reading documentation. Here are a few links to get you
started:

- <https://nixos.org/learn>
- <https://nix.dev/>
- <https://zero-to-nix.com/>

The links above have a lot of jumping off points to other resources. Let me know
if I left anything out.

The [nix-community impermanence
readme](https://github.com/nix-community/impermanence) is a must-read unless you
decided to not use:

```nix
# flake.nix
{
  inputs = {
    # ...
    impermanence.url = "github:nix-community/impermanence";
  };
  # ...
}
```

## Finding optimal alignment

When using `parted`, a simple way around alignment issues is to use percentages
or MB / GB units. When this is done, `parted` effectively searches a calculated
distance around the given location to find optimal alignment. However, I like to
have my partitions in nice clean power of 2 IEC unit sizes (e.g. MiB) because
that's what most tools default to (e.g. `gparted`, `cfdisk`, `lsblk`, `df -h`
etc.). The problem is when using IEC units or exact sectors, `parted` will not
search for optimal alignment. It basically assumes you know what you're doing.
[^7] [^8] [^9]

What we need to find is the starting sector of the first partition. A simple
trick to do this with `parted` is to:

```sh
sudo parted $DISK --script \
  mklabel gpt \
  mkpart primary 0% 100% \
  unit MiB print \
  rm 1
```

This will print out something like the following. Note the `Start`.

```
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 40960MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start    End       Size      File system  Name     Flags
 1      1.00MiB  40959MiB  40958MiB               primary
```

Now wipe the device to start over:

```sh
sudo wipefs -a $DISK
```

Head back to the "Partitioning" section and adjust the values for your `Start`.

## Harmless error on shutdown

When shutting down or rebooting, you might see a flash of red in the logs:

```
[FAILED] Failed unmounting /nix.
```

Seems to be related to [this
issue](https://github.com/nix-community/impermanence/issues/21). Fortunately, it
seems to be harmless and doesn't cause any delay.

If you want to see the error more clearly, try `sudo halt`.

---

[^1]: [Nixos Wiki Impermanence](https://nixos.wiki/wiki/Impermanence)

[^2]: [Nix Community Impermanence](https://github.com/nix-community/impermanence)

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
running across. It should be pretty straight forward to remove swap from the
script snippets above.

[^6]: This blog post is already taking too long. Making it "opinionated" makes it
    easier to write.

[^7]: [Archlinux Parted Alignment](https://wiki.archlinux.org/title/Parted#Alignment)

[^8]: [HQ Code Shop Blog Post on GNU Parted](https://blog.hqcodeshop.fi/archives/273-GNU-Parted-Solving-the-dreaded-The-resulting-partition-is-not-properly-aligned-for-best-performance.html)

[^9]: [Unix StackExchange Question on Parted Alignment](https://unix.stackexchange.com/questions/38164/create-partition-aligned-using-parted/401118#401118)
