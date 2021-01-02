# omv-zfs-ras-nas
Documentation of my [openmediavault](https://www.openmediavault.org/) [OpenZFS](https://github.com/openzfs/zfs) [Raspberry Pi 4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/) [Network-attached storage](https://en.wikipedia.org/wiki/Network-attached_storage) setup.

[![CC BY-SA 4.0][cc-by-sa-shield]][cc-by-sa]


## Why?

- I like OMV
- I want a system that I can run 24/7 with minimal power consumption.
- RAID is not supported by OMV when the hard disks are connected via USB. Technically using RAID with USB disks is possible but unreliable and dangerous.
- ZFS has some awesome features:
  - Mirroring (replaced RAID)
  - Snapshots
  - Smart behavior when recovering from a failure (only syncs real data, not the entire drive)
  - Supports volumes (replaces LVM)
  - Supports encryption (replaces LUKS)

## My Rig

- Raspberry Pi 4 with 4GB RAM.
- A case with a fan.
  - Initially connected to 5V during compilation and testing of ZFS.
  - Afterwards connected to 3,3V to avoid noise.
- Two 2 TB laptop hard-drives selected for especially low energy consumption.
- An extra USB power supply with two USB ports to supply the hard-drives with enough power, especially during spin-up.
- The hard-drives are connected to the Raspberry Pi and the extra USB power supply using USB Y-cables and two cheap USB-SATA adapters.

## Reading Material

- [OpenZFS - Building ZFS](https://openzfs.github.io/openzfs-docs/Developer%20Resources/Building%20ZFS.html)
- [NAS-Server mit Raspberry Pi und OpenMediaVault einrichten](https://www.heise.de/download/blog/NAS-Server-mit-Raspberry-Pi-und-OpenMediaVault-einrichten-3468200) (in german)
- [Raspberry Pi 4: Minimalistisches NAS mit ZFS-Dateiverwaltung selbst bauen](https://www.heise.de/ratgeber/Raspberry-Pi-4-Minimalistisches-NAS-mit-ZFS-Dateiverwaltung-selbst-bauen-4927272.html?seite=all) (behind paywall, in german)
- [Introducing Raspberry Pi Imager, our new imaging utility](https://www.raspberrypi.org/blog/raspberry-pi-imager-imaging-utility/)

## Download Locations

- [Raspberry Pi Imager](https://www.raspberrypi.org/software/)
- [64-Bit-Version von Raspberry Pi OS](https://www.heise.de/news/64-Bit-Version-von-Raspberry-Pi-OS-4771111.html) (in german)

## Setup Steps

### 1. Install Raspberry Pi OS to microSD Card

Install the [Raspberry Pi Imager](https://www.raspberrypi.org/software/) software and install **Raspberry Pi OS Lite** (32-bit OS) to a microSD card.

Alternatively you can try out the new but not yet officially released 64-bit version of Raspberry Pi OS. You can install [Raspberry Pi OS 64-bit images](https://downloads.raspberrypi.org/raspios_arm64/images/) using the Raspberry Pi Imager as well but you have to download them manually first.

The ZFS filesystem is intended to be run on 64-bit operating systems but also works on 32-bit systems, albeit with reduced performance.

### 2. Update your Raspberry Pi and your Raspberry Pi OS

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

### 4. Install OMV

Download the installer script:

```bash
wget -O install-omv.sh https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install
```

Run the installer with the `-n` switch to skip the network setup:

```bash
sudo bash install-omv.sh -n
```

OMV's network setup leads to non-critical errors in debian's network scripts. I did not look into that matter and simply disabled OMV's network setup since I do not need OMV to manage the Raspberry Pi's network interface and I like my start-up sequence free of error messages.

[cc-by-sa]: http://creativecommons.org/licenses/by-sa/4.0/
[cc-by-sa-shield]: https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg
