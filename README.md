# omv-zfs-ras-nas
Documentation of my [openmediavault](https://www.openmediavault.org/) [OpenZFS](https://github.com/openzfs/zfs) [Raspberry Pi 4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/) [Network-attached storage](https://en.wikipedia.org/wiki/Network-attached_storage) setup.

[![CC BY-SA 4.0][cc-by-sa-shield]][cc-by-sa]


## Why?

- I like OMV
- I want a system that I can run 24/7 with minimal power consumption.
- RAID is not supported by OMV when the hard disks are connected via USB. Technically using RAID with USB disks is possible but unreliable and dangerous.
- ZFS has some awesome features:
  - Supports (among other RAID-like modes) mirroring (replaces mdadm).
  - Supports snapshots.
  - Supports volumes (replaces LVM).
  - Supports encryption (replaces LUKS).
  - Supports compression.
  - ZFS can directly transfer snapshots to other machines, thus facilitating easy and fast off-site backup strategies.
  - Smart behavior when recovering from a failure (only syncs real data, not the entire drive).

## Disclaimer

RAID is not a backup, it only protects you from a hard-drive failure. ZFS' support for snapshots adds some limited backup functionality. I still recommend a regular off-site backup.

## Hardware

- Raspberry Pi 4 with 4GB RAM.
- A case with a fan.
  - Initially connected to 5V during compilation and testing of ZFS.
  - Afterwards connected to 3,3V to avoid noise.
- Two 2 TB laptop hard-drives selected for especially low energy consumption.
- An extra USB power supply with two USB ports to supply the hard-drives with enough power, especially during spin-up.
- The hard-drives are connected to the Raspberry Pi and the extra USB power supply using USB Y-cables and two cheap USB-SATA adapters.

## Unsolved Problems:

The cheap USB-SATA adapters have a JSM (JMicron) Chipset which cannot properly handle some hdparm/smartctl commands. Configuring power management seems to work but querying whether the drive has spun down does not work. This currently prevents me from making the device spin down because it is woken up regularly for snapshots.

Reading material:
- [Smartmontools USB Device Support](https://www.smartmontools.org/wiki/Supported_USB-Devices)
- [Smartmontools USB Device Support (unsupported devices)](https://www.smartmontools.org/wiki/Unsupported_USB-Devices)

The chipset ASM1351 seems to support the required commands but I have not found a product that uses this chipset and meets my other requirements.

## Benchmarks:

Sending a tar archive from a powerful machine to the ras-nas machine to store it on a ZFS filesystem:

```bash
ras-nas:~# nc -l 8888 > {dest on zfs}

nas:~# tar --acl -cpf - -C /srv/dev-disk-by-uuid-{uuid} . | pv -trabpte | nc -q0 ras-nas 8888
       342GiB 2:08:44 [45.4MiB/s] [45.4MiB/s]
```

Sending a snapshot from the ras-nas machine to a powerful machine:

```bash
nas:~# nc -l 8888 | zfs receive {dest zfs filesystem}

ras-nas:~# zfs send {source zfs filesystem} | pv -trabpte | nc -q0 nas 8888
           341GiB 1:08:36 [85.0MiB/s] [85.0MiB/s]
```

## Reading Material

- [OpenZFS - Building ZFS](https://openzfs.github.io/openzfs-docs/Developer%20Resources/Building%20ZFS.html)
- [NAS-Server mit Raspberry Pi und OpenMediaVault einrichten](https://www.heise.de/download/blog/NAS-Server-mit-Raspberry-Pi-und-OpenMediaVault-einrichten-3468200) (in german)
- [Raspberry Pi 4: Minimalistisches NAS mit ZFS-Dateiverwaltung selbst bauen](https://www.heise.de/ratgeber/Raspberry-Pi-4-Minimalistisches-NAS-mit-ZFS-Dateiverwaltung-selbst-bauen-4927272.html?seite=all) (behind paywall, in german)
- [Introducing Raspberry Pi Imager, our new imaging utility](https://www.raspberrypi.org/blog/raspberry-pi-imager-imaging-utility/)
- [64-Bit-Version von Raspberry Pi OS](https://www.heise.de/news/64-Bit-Version-von-Raspberry-Pi-OS-4771111.html) (in german)
- [NAS with ZFS on Raspberry Pi 4](https://www.nasbeery.de/)

## Setup Steps

### 1. Install Raspberry Pi OS to microSD Card

Install the [Raspberry Pi Imager](https://www.raspberrypi.org/software/) software.

Click on "Choose OS", then "Raspberry Pi OS (**other**)", then select "Raspberry Pi OS **Lite** (**64-bit**)"

The ZFS filesystem is intended to be run on 64-bit operating systems but also works on 32-bit systems, albeit with reduced performance.

### 2. Update your Raspberry Pi and your Raspberry Pi OS

To avoid locale warnings you should run `sudo raspi-config` right away and select the locales of your choice, and most importantly, select a default locale, then reboot.

To update the OS, run:

```bash
$ sudo apt update
$ sudo apt full-upgrade
$ sudo apt autoremove
$ sudo apt autoclean
$ sudo reboot
```

To see if updates for the EEPROM are available, run:

```bash
$ sudo rpi-eeprom-update
```

To actually perform an update of the EEPROM, run at *your own risk*:

```bash
$ sudo rpi-eeprom-update -a
$ sudo reboot
```

For further information read [Raspberry Pi 4 boot EEPROM](https://www.raspberrypi.org/documentation/hardware/raspberrypi/booteeprom.md).

### 3. Configure Raspberry Pi OS

This step depends on your setup and your needs. For the goal of installing OMV and ZFS no special configuration steps are required.

To configure your Raspberry Pi OS run:

```bash
$ sudo raspi-config
```

**Recommendations:**

These are just my personal preferences:

- Enable SSH in `raspi-config`.
- Use `ssh-copy-id` to transfer your public SSH keys to the Raspberry Pi.
- Disable root login and password authentication in `/etc/ssh/sshd_config`.
- Install `tmux` for remote work on the Raspberry Pi.
- Enable mouse support in tmux:
  - Install `gpm`.
  - Add `set -g mouse on` to ~/.tmux.conf (you probably have to create it first).
- Install `glances`, and `dstat` for system performance measurements. `glances` is not available in debian, use pip for installation instead: `sudo apt install python3-pip && sudo pip3 install glances`
- Disable swap:
```bash
sudo systemctl stop dphys-swapfile.service
sudo systemctl disable dphys-swapfile.service
```

### 4. Install OMV

Download the installer script:

```bash
wget -O install-omv.sh https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install
```

OMV's network setup leads to non-critical errors in debian's network scripts. I did not look into that matter and simply disabled OMV's network setup since I do not need OMV to manage the Raspberry Pi's network interface and I like my start-up sequence free of error messages.

Run the installer with the `-n` switch to skip the network setup:

```bash
sudo bash install-omv.sh -n
```

Do not configure OMV just yet.

### 5. Install ZFS

Nowadays, ZFS can be installed from backports (bullseye):

```bash
echo "deb http://deb.debian.org/debian bullseye-backports main" | sudo tee -a /etc/apt/sources.list
sudo apt update
sudo apt install raspberrypi-kernel-headers
sudo apt install -t bullseye-backports zfsutils-linux zfs-dkms zfs-zed
```

Test whether the ZFS module can be loaded:

```bash
$ modprobe zfs
$ lsmod | grep zfs
```

The output of the last command should look something like this:

```
zfs                  3461120  6
zunicode              331776  1 zfs
zzstd                 430080  1 zfs
zlua                  147456  1 zfs
zcommon                98304  1 zfs
znvpair                90112  2 zcommon,zfs
zavl                   16384  1 zfs
icp                   294912  1 zfs
spl                   106496  7 znvpair,zcommon,zfs,icp,zzstd,zlua,zavl
```

Make sure the ZFS module is loaded at boot:

```bash
$ echo "zfs" | sudo tee -a /etc/modules
```

Enable ZFS services at boot:

```bash
$ sudo systemctl enable zfs-import-cache.service zfs-import.target zfs-mount.service zfs-zed.service zfs.target
```

If you do not plan to install the OMV ZFS plugin, you may want to manually setup zfs-zed in `/etc/zfs/zed.d/zed.rc`.
Otherwise you can leave this file alone for now.

### 6. Install the OMV ZFS plugin

This step requires the installation of kernel headers for our system. This happened in the previous step (5. Install ZFS) already.

In the OMV web frontend, go to `System` -> `Plugins` and install `openmediavault-zfs` (not the `armhf` version).

### 7. Create ZFS Pools and Filesystems

Now one can create ZFS pools and filesystems through the OMV admin interface. Personally I prefer to do this on the console:

```bash
$ sudo zpool create -O acltype=posixacl -O xattr=sa -O atime=off -O dnodesize=auto -O compression=lz4 -o ashift=12 tank mirror /dev/sda /dev/sdb
```

- The attributes `acltype` and `xattr` allow us to use ACLs on the filesystems.
- I have read somewhere that `dnodesize=auto` is good for your karma, so I have added it here.
- The attribute `ashift` has to be set to your drive's sector size. It is the exponent in 2^12 = 4096 = 4K sector size. Adapt it to your needs.
- Set the compression to lz4 (default would be: none).
- The pool is called "tank" and is a mirror setup consisting of my two hard-drives `sda` and `sdb`.

Every pool automatically comes with a root filesystem that will be mounted at `/tank`. I recommend not using that and instead create individual filesystems for your needs. For example:

```bash
sudo zfs create tank/homes
sudo zfs create tank/shared
```

These filesystems will inherit the settings we used during pool creation but can be customized (except for the pool-specific settings).

To make sure that the pool is imported on system startup we have to make sure that a `cachefile` exists for our pool:

```bash
$ sudo zpool set cachefile=/etc/zfs/zpool.cache tank
```

This file is read by the systemd service `zfs-import-cache.service` which will then import the corresponding pools. Afterwards the systemd service `zfs-mount.service` will mount all filesystems found in the imported pools.

### 8. Automatic Snapshots

**Warning:** Enabling automatic snapshots probably prevents your drives from spinning down *ever*, though I have not verified this.

To enable automatic snapshots at various intervals, do the following:

```bash
$ wget -O zfs-auto-snapshot.tar.gz https://github.com/zfsonlinux/zfs-auto-snapshot/archive/upstream/1.2.4.tar.gz
$ tar -xzf zfs-auto-snapshot.tar.gz
$ cd zfs-auto-snapshot-upstream-*
$ sudo make install
```

Change the number of kept snapshots to match your personal preferences:

```bash
sudo sed -i 's/keep=24/keep=48/g' /etc/cron.hourly/zfs-auto-snapshot
sudo sed -i 's/keep=12/keep=3/g' /etc/cron.monthly/zfs-auto-snapshot
```

[cc-by-sa]: http://creativecommons.org/licenses/by-sa/4.0/
[cc-by-sa-shield]: https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg

## TODOs

- Use different device names:
```
zpool export <pool-name>
zpool import -d /dev/disk/by-partuuid/<uuid> /dev/disk/by-partuuid/<uuid>
zpool import <pool-name>
```
- Import all available pools:
```
zpool import -a
```
- Get rid of unnecessary alerts on reboot
```
nano /etc/default/openmediavault  # Add or edit line: OMV_MONIT_DELAY_SECONDS="15" until alerts are not longer sent
omv-salt stage run prepare
omv-salt deploy run monit
```
- Close tmuxinator session: `Ctrl+b &`
- List all zfs snapshots: `zfs list -t snapshot`
