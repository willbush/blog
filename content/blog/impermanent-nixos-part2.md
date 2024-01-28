+++
title = "Impermanent NixOS - Part 2: on a Framework Laptop + LUKS + flakes + hyprland"
date = 2024-01-27
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

Write the ISO to a USB drive:

```sh
sudo dd if=<path-to-iso> of=/dev/sdX status=progress bs=4M conv=fsync
```

Boot from the USB drive to get into the live NixOS environment.

For me `lsblk` reveals my nvme drive is `/dev/nvme0n1`:

```sh
export DISK=/dev/nvme0n1
```

I have a previous installation of NixOS on the drive that I want to wipe out:

```sh
sudo wipefs -a $DISK
```
