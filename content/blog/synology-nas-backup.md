+++
title = "Synology NAS Backups with Minimal Bus Factor"
description = "Synology NAS Backups with Minimal Bus Factor and Maximum Complexity"
date = 2023-03-27

[taxonomies]
categories = ["Tech"]
tags = ["tools", "linux"]

[extra]
toc = true
cc_license = true
+++

If you're wanting to backup your Synology NAS, you probably just want to use one
of their off-the-shelf solutions in their Package Center. However, if you're
stubborn like me and want to use the same CLI tool you use elsewhere, this post
is for you. I'll describe how to use [restic](https://restic.net/) to backup
folders you care about on your Synology NAS to an external hard drive and S3
conformant cloud storage providers that are [supported by
restic](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#preparing-a-new-repository)
and one supported by [rclone](https://rclone.org/). If you want to use another
command line tool, much of the instructions I provide here should still be
applicable.

<!-- more -->

# Motivation

## Raid redundancy is not a backup

Hard drives installed in a NAS run under the same conditions, workloads, and are
often the same brand and bought at the same time. For those reasons, multi-drive
failure is a real possibility. Not to mention, fire, flood, theft, and other
disasters.

## Not caring until it's gone

I lost some videos I recorded of myself playing videos games a long time ago
that I very much wish I still had for nostalgia. At the time I ran 2 10k RPM WD
Raptors in RAID 0 in the early 2000s — not thinking about doubling the risk of
data loss. Of course, one of the drives failed, and I had no backups.

I have been "backing up" since then, but not until I read about [3-2-1 Backup
Rule](https://www.vmwareblog.org/3-2-1-backup-rule-data-will-always-survive/),
did I realize I need to up my game.

# My imperfect backup solution

I really only have ~90GB of data that I never want to lose — Family photos,
videos, docs etc. — out of several terabytes of data I wouldn't lose sleep over.
Or put another way, I only want to pay for what I need. However, there's no
reason this won't scale to terabytes of data. The only limiting factor is your
upload speed and patience.

I ditched Dropbox for [Syncthing](https://syncthing.net/) and use that to keep a
[bunch of
things](https://github.com/willbush/system/blob/15bc0c058db804abb7633f70aaa4dd430fcc796f/modules/services/syncthing.nix)
in sync across many devices. I like to tell myself this is enough redundancy
until I perform a manual backup — famous last words.

Once a year I offload from the syncthing folders to encrypted folders on the
NAS. Some of these folders fall into the "don't want to lose" category. Then I
perform a manually backup using `restic` to an external hard drive and a few
cloud providers. Multiple cloud providers is my own compensation for the lack of
not using a real 2nd media type as 3-2-1 suggests. I put one of the cloud
backups in a region on the other side of the planet just in case a [Carrington
like event](https://youtu.be/mJCytV7PUzk) or even [100X
worse](https://youtu.be/EXstRq3vius) happens. Not saying I think that will cause
massive data loss, but it's fun to be paranoid.

Restic snapshots are encrypted with a password. Of course, I generate long
complex passwords for each backup destination. Therefore, I'm not concerned
about if the cloud provider supports encryption at rest or E2E encryption.

## Backing up passwords and encryption keys

All these redundant backups are only as good as storage of secrets.

I use [gopass](https://www.gopass.pw/) to manage my passwords which is a
derivative of [pass](https://www.passwordstore.org/). In short, the passwords
are encrypted and stored in a git repo. I have this repo on multiple devices so
I get the distributed redundancy from git. I could improve this by having auto
sync to another git forge provider though.

The [Android
app](https://play.google.com/store/apps/details?id=dev.msfjarvis.aps&hl=en_US&gl=US)
works better than you'd expect from a less popular password manager. However,
bootstrapping things with GPG keys is a huge pain compared to other password
managers. I used to sync a KeePass database with Syncthing.

I haven't got my wife to switch from KeePass yet. Definitely, a trade-off in
complexity. Complex to setup, but easier to use and manage. I love using Emacs
to manage my passwords. I haven't looked into it much, but I wish I could easily
sync between gopass and something like Bitwarden for the best of both worlds.

## Bus factor of 1

So what if I get hit by a bus? Can my significant other figure out this
convoluted mess? This is a real concern. Documenting my backup strategy is a
start.

What if I cancel my credit card and forget to update the payment method for the
cloud providers? I don't know. I just checked and the providers I choose don't
have a way to add a backup payment method. So I guess I'll just use different
credit cards for the 2nd paid providers I'm using and hope for the best.

The intersection of security, redundancy, and not getting locked in by
proprietary tools makes the bus factor hard problem to solve.

# Cloud providers and costs

I've been using Storj and Backblaze B2 since 2021. I picked Storj because the
first 150GB is free — more than I need for now, but not something I can depend
on.

Last year, 2022, Backblaze B2 cost for me was $3.20 for the year. About $0.27
per month after tax.

~~This year I added Wasabi because of it's S3 conformance, the backup steps are
nearly exactly the same, and low cost.~~ - edit (2023-08-12) actually after some
time using the service I found it to be much more expensive than Blackblaze B2
~$6 / month. So I wouldn't recommend it if your use case is similar to mine.

# Hardware

## External Hard Drive

On Ebay, in late 2021, I bought a:

- ~$55 WD My Passport 4TB Certified Refurbished Portable Hard Drive.

Since using the space doesn't incur any additional cost like cloud storage, I
backup more than just the "don't want to lose" category.

## Synology Diskstation

- Model: DS918+
- Software Version: 7.1.1-42962 Update 4
- RAM: 16GB
- CPU: INTEL Celeron J3455
- HDD: 4x HGST Deskstar NAS 6TB 7200 RPM

# Synology is not for terminal junkies

I bought a Synology NAS because at the time I didn't want to roll my own. Trying
to do things in the terminal on a Synology NAS is a PITA for reasons that will
become clear shortly. I will definitely be rolling my own in the future.

# Preliminaries

## Install Docker

Docker is why this process is more of a pain than it should be. In 2021, I was
able to `ssh` into my NAS and install the [nix package
manager](https://nixos.org/download.html) and then install
[restic](https://search.nixos.org/packages?channel=unstable&from=0&size=50&sort=relevance&type=packages&query=restic).

It's not clear to me why I no longer have `nix`. Perhaps an update blew it away?
And I'm no longer able to install it. Both the official and
[unofficial](https://zero-to-nix.com/) installers fail at least for me at the
time of writing.

Synology's distribution of Linux doesn't appear to have any other package
managers available, but we can fall back on Docker.

Installing Docker should be easy enough through the Package Center.

## Enable SSH

- [offical docs](https://kb.synology.com/en-id/DSM/tutorial/How_to_login_to_DSM_with_root_permission_via_SSH_Telnet)
- [unofficial blog post](https://mariushosting.com/how-to-ssh-into-a-synology-nas/)

If you follow the instructions on the Synology website, you should be able to
`ssh` into your NAS with an admin account.

For me this looks like:

```sh
ssh will@nas -p 6666
```

In a strange act of security through obscurity, I didn't use the standard port 22
for SSH.

## Set TERM if needed

If `echo $TERM` is not one of the following values, such as `xterm`, then your
probably going to have issues with the terminal (such as backspace not working).

```sh
root@NAS:~# find /usr/share/terminfo/ -type f
/usr/share/terminfo/a/ansi
/usr/share/terminfo/x/xterm-256color
/usr/share/terminfo/x/xterm
/usr/share/terminfo/x/xterm-color
/usr/share/terminfo/d/dumb
/usr/share/terminfo/s/screen
/usr/share/terminfo/s/screen-bce
/usr/share/terminfo/s/screen-256color
/usr/share/terminfo/s/screen-256color-bce
/usr/share/terminfo/v/vt100
/usr/share/terminfo/v/vt52
/usr/share/terminfo/v/vt102
/usr/share/terminfo/v/vt220
/usr/share/terminfo/u/unknown
/usr/share/terminfo/l/linux
```

If you find yourself unable to backspace, `ctrl-C` to intreupt the command and
then run:

```sh
TERM=xterm
```

or

```sh
TERM=xterm-256color
```

And that will probably fix it.

I use [alacritty](https://github.com/alacritty/alacritty) and many servers I SSH
into don't have terminfo for it. Setting the TERM variable to `xterm-256color`
prepended to `ssh` is another way to get around this. However, I'm not sure if this works
from Windows. For example:

```sh
TERM=xterm-256color ssh will@nas -p 6666
```

## Root

Most of the examples from now on assume you are logged in as root.

I will use `root@NAS:~#` to denote commands run as root.

Login with root with the same password as your admin account:


```sh
sudo -i
```

## Fix bash history

Or expect to lose it.

```sh
root@NAS:~# echo $HISTFILE
/var/tmp/.bash_history
root@NAS:~# echo $HISTSIZE
100
root@NAS:~# echo $HISTFILESIZE
100
```

Ha! I wish I checked these variables earlier. No! Instead I lost my bash
history, and then increased the history size. Lost it again and realized the
history file was in a `tmp` directory.

Increase your history size and set the history file to a permanent location:

```sh
root@NAS:~# echo "export HISTSIZE=10000" >> ~/.bashrc
root@NAS:~# echo "export HISTFILESIZE=10000" >> ~/.bashrc
root@NAS:~# echo "export HISTFILE=/root/.bash_history" >> ~/.bashrc
```

## Format external hard drive

Ok, so I formatted my external hard drive from my desktop in Linux. I wasn't
thinking about making a blog post at the time. It should be doable from the
Synology. For me, `fdisk` and `mkfs.ext4` are available, but not `lsblk`.

I'm going to do my best to estimate the steps for formatting an external hard as
root on a Synology NAS, but there's good chance I'm missing something:

Plug in the drive and print out the list of devices using `fdisk -l`:

For me I see `/dev/sdq` is my external hard drive:

```
Disk /dev/sdq: 3.7 TiB, 4000752599040 bytes, 7813969920 sectors
Disk model: My Passport 25E2
...
```

I can also see that it's auto-mounts to `/volumeUSB1/usbshare`:

```
root@NAS:~# cat /proc/mounts | grep -i sdq
/dev/sdq /volumeUSB1/usbshare ext4 rw,relatime,nodelalloc,synoacl,data=ordered 0 0
```

If your external hard drive doesn't have a filesystem yet, then it probably won't auto-mount.

Unmount the external hard drive if needed:

```sh
root@NAS:~# umount /volumeUSB1/usbshare/
```

Wipe the filesystem and create a new `ext4`:

```sh
root@NAS:~# wipefs -a /dev/sdx
root@NAS:~# mkfs.ext4 /dev/sdx
```

Replace `sdx` above with your device name.

You should now be able to unplug and plug it back in for to auto-mount to
`/volumeUSB1/usbshare` or something similar.

# Docker basics

Let's start with passing `--help` to `restic`:

```sh
docker run -it --rm restic/restic --help
```

Here's a ChatGPT breakdown of the command:

> This command is used to run a Docker container with the restic/restic image,
> which is an open-source backup program. Let's break down the command and its
> components:
>
> 1. `docker run`: This is the primary command to run a new container from a
>    Docker image.
>
> 2. `-it`: This is a combination of two flags:
>
>    - `-i` (interactive): Keeps the standard input (stdin) open to interact
>      with the container.
>
>    - `-t` (tty): Allocates a pseudo-TTY (terminal) to the container, allowing
>      you to interact with the container as if it were a standard terminal.
>
> 3. `--rm`: This flag removes the container automatically when it exits. This
>     is useful for keeping your system clean and avoiding unnecessary container
>     accumulation.
>
> 4. `restic/restic`: This is the name of the Docker image that the container
>      will be based on. In this case, it's the official Restic backup program's
>      image.
>
> 5. `--help`: This is an argument passed to the Restic program inside the
>      container. It tells Restic to display its help message, which contains
>      information about the available commands and options.

This is a good starting point for understanding what's going on in the rest of
the blog post. I'll often break long commands across multiple lines using `\` at
the end of the line. Usually I edit these commands outside of the shell and
paste them in with `ctrl-shift-v`.

# Let's backup!

## Things to note

For me Synology has all my data under `/volume1`.

And the root mount `/` has very little space:

```
root@NAS:~# df -h
Filesystem              Size  Used Avail Use% Mounted on
/dev/md0                2.3G  1.8G  431M  81% /
...
```

In fact, not enough for what `restic` is going to need for cache: `~/.cache/restic/`

Therefore, to avoid running out of cache space, I use the `--cache-dir` to
change the cache directory to a place that has much more space available.

The commands I give below are mostly self-contained. However, note that one
could just change the entrypoint of the container to `sh`

```sh
root@NAS:~# docker run -it --rm \
  -v /volume1:/volume1 \
  --entrypoint /bin/sh \
  restic/restic
```

And then you're in a shell inside the container where you can run `restic`
commands directly.

## External hard drive

Initialize the restic repository if not done yet.

```sh
root@NAS:~# docker run -it --rm \
  -v /volumeUSB1/usbshare:/usbshare \
  restic/restic \
  -r /usbshare/restic-backups/ \
  init
```

It will prompt you for a password to encrypt the repo. Make sure you keep it
safe.

Note that `-v /volumeUSB1/usbshare:/usbshare` mounts the external hard drive
volume to `/usbshare` inside the container.

The `restic` repo is now initialized at
`/volumeUSB1/usbshare/restic-backups/` (outside the container).

And backup:

```sh
root@NAS:~# docker run -it --rm \
  -v /volume1:/volume1 \
  -v /volumeUSB1/usbshare:/usbshare \
  restic/restic \
  -r /usbshare/restic-backups/ \
  --cache-dir /volume1/.cache/restic/ \
  --verbose \
  backup \
  /volume1/backup-safe/ \
  /volume1/backup/ \
  /volume1/games/ \
  /volume1/music/ \
  /volume1/photo/ \
  /volume1/syncthing/
```

Replace `/volume1/*` folders with what you want to backup.

Again note that I am mounting `/volume1` to `/volume1` inside the container.

## Backblaze B2

Also see: <https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html#backblaze-b2>

Initialize the restic repository if not done yet:

```sh
root@NAS:~# docker run -it --rm \
  -e B2_ACCOUNT_ID='<MY_APPLICATION_KEY_ID>' \
  -e B2_ACCOUNT_KEY='<MY_APPLICATION_KEY>' \
  restic/restic \
  -r b2:<B2-BUCKET-NAME>:restic-backups/ \
  init
```

Note `-e` is a way to pass environment variables to the container.

Backup:

```sh
root@NAS:~# docker run -it --rm \
  -e B2_ACCOUNT_ID='<MY_APPLICATION_KEY_ID>' \
  -e B2_ACCOUNT_KEY='<MY_APPLICATION_KEY>' \
  -v /volume1:/volume1 \
  restic/restic \
  -r b2:<B2-BUCKET-NAME>:restic-backups/ \
  --cache-dir /volume1/.cache/restic/ \
  --verbose \
  backup \
  /volume1/backup-safe/ \
  /volume1/photo/
```

## Wasabi

{% note(header="Note") %}

I no longer use Wasabi due to higher costs, but I'll leave this here incase you
want to use it.

{% end %}

Also see: <https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html#wasabi>

Initialize the restic repository if not done yet:

```sh
root@NAS:~# docker run -it --rm \
  -e AWS_ACCESS_KEY_ID='<YOUR-WASABI-ACCESS-KEY-ID>' \
  -e AWS_SECRET_ACCESS_KEY='<YOUR-WASABI-SECRET-ACCESS-KEY>' \
  restic/restic \
  -r s3:<WASABI-SERVICE-URL>/<WASABI-BUCKET-NAME> \
  init
```

Note `-e` is a way to pass environment variables to the container.

Backup:

```sh
root@NAS:~# docker run -it --rm \
  -e AWS_ACCESS_KEY_ID='<YOUR-WASABI-ACCESS-KEY-ID>' \
  -e AWS_SECRET_ACCESS_KEY='<YOUR-WASABI-SECRET-ACCESS-KEY>' \
  -v /volume1:/volume1 \
  restic/restic \
  -r s3:<WASABI-SERVICE-URL>/<WASABI-BUCKET-NAME> \
  --cache-dir /volume1/.cache/restic/ \
  --verbose \
  backup \
  /volume1/backup-safe/ \
  /volume1/photo/
```

Replace `/volume1/*` folders with what you want to backup.

## Storj

Ok fair warning, this is more involved than the other ones. In fact I almost
gave up this year and it was part of the reason I looked into another paid
storage provider. On top of that, I failed to write down the steps as I did
them, so I'm going to estimate again from memory. Here is the documentation I followed:

- [backup-with-restic](https://docs.storj.io/dcs/how-tos/backup-with-restic)
- [rclone-with-hosted-gateway](https://docs.storj.io/dcs/how-tos/sync-files-with-rclone/rclone-with-hosted-gateway)
- <https://rclone.org/storj/>

I'm using their "S3 compatible hosted gateway" integration pattern to minimize upload.

They offer end-to-end encryption, but I opted for server-side encryption because
the data that I'm backing up there is already encrypted using restic.

This is going to require two tools `rclone` and `restic`.

So I used the `restic` docker image and changed the entrypoint to `sh`.


```sh
root@NAS:~# docker run -it --rm \
  -v /volume1:/volume1 \
  --entrypoint /bin/sh \
  restic/restic
```

Once you're inside the running container, you can use the `apk` package manager to install `rclone`:

```sh
apk add rclone
```

Using [this
documentation](https://docs.storj.io/dcs/how-tos/sync-files-with-rclone/rclone-with-hosted-gateway)
above, perform `rclone config` and follow the steps in the docs.

Once complete, Use the `mkdir` command to create new bucket, e.g. `mybucket`.

```sh
rclone mkdir storj:mybucket
```

Again, still in the container, perform the backup:

```sh
restic -r rclone:storj:<STORJ-BUCKET-NAME>/restic-backups \
  --cache-dir /volume1/.cache/restic/ \
  --verbose \
  backup \
  --pack-size=60 \
  /volume1/backup-safe/ \
  /volume1/photo/
```

Replace `/volume1/*` folders with what you want to backup.

# Restore from External HDD Backup

Ok this will be incomplete. I'll try to update this next year when I do backups
again.

This year I was doing my taxes and had a bit of a fire drill when I couldn't
find any previous years. I later realized I simply accidentally dragged them into
a random folder using Synology's web UI.

Here are some rough steps it took to mount the backup and be able to view the
files from Linux:

```sh
$ lsblk # find the device name of the external HDD
$ sudo mkdir /mnt/media
$ sudo mount -t ext4 -o ro /dev/sdx /mnt/media
$ sudo restic -r /mnt/media/restic-backups/ mount ~/backup-mount \
  --allow-other --no-default-permissions
$ cd ~/backup-mount && ls -lah # view files
$ sudo umount /mnt/media # unmount when done.
```

# Cleanup

It's worth noting you should search and delete passwords and keys from your
`history` when done. Since most of everything was done in Docker, there
shouldn't be much to clean up.

You can find your history file with `echo $HISTFILE`.
