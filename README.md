# crypto-manage – OpenWrt LUKS Crypto Manager

A lightweight, POSIX-compliant shell script for **OpenWrt** (BusyBox) that orchestrates the automatic opening, mounting, unmounting, and closing of LUKS-encrypted hard drives and containers.

The script reads the configuration natively from the OpenWrt system (`UCI`) and features built-in data loss protection via process blockade checking (`fuser`) as well as a simulated dry-run mode (`Dry-Run`)

---
## 1) Introduction & Concept

OpenWrt provides excellent crypto support via packages like `cryptsetup`, but by default lacks an automated, dynamic system to cleanly manage encrypted hard drives during runtime via scripts. Manually triggering `cryptsetup open`, creating mount points, mounting via the correct device mapping, and later closing cleanly (without destroying open file handles) is error-prone and cumbersome.

`crypt-manage` closes this gap. It acts as an intelligent link between OpenWrt's configuration interface (UCI) and the Linux system tools. 

**The core advantages:**
* **Centralized:** It uses the native OpenWrt configuration files (`/etc/config/cryptsetup` and `/etc/config/fstab`). No redundant paths in the script.
* **Safe (Safety First):** Before closing an encrypted partition, it checks whether processes are still accessing the disk. This prevents hard data loss.
* **Dry-Run:** Every command is only *simulated* by default. The action is only armed by explicitly appending the `go` parameter.
---

## 2) Prerequisites
### System Dependencies and USB Support
To enable your OpenWrt router to detect, encrypt, and mount USB storage media in the first place, the following packages must be installed.
First, update the package lists:
```
apk update
```
Install the drivers for USB support and USB storage media (USB Mass Storage):
```
apk add kmod-usb-storage kmod-usb-storage-uas block-mount
```
Install the required crypto packages for LUKS encryption:
```
apk add cryptsetup fuser kmod-crypto-xts
```
Install the drivers for the desired filesystem (using ext4 as an example here):
```
apk add kmod-fs-ext4
```
(Note: uci, mount, umount, and awk are part of BusyBox by default and are already present on the system).

### Basic Understanding (Important!)
This script is not intended to create a new encrypted hard drive. To ensure the security of your data, you must manually set up the crypto filesystem once in advance. This ensures that you understand the encryption and keying process (passphrase).

### Quick Guide: Preparing a LUKS Disk Manually
1. **Encrypt the hard drive (formatting):**
```
cryptsetup luksFormat /dev/sda1
```
2. **Open for testing:**
```
cryptsetup open /dev/sda1 mein_safe
```
3. **Create a filesystem (e.g., ext4):**
```
mkfs.ext4 /dev/mapper/mein_safe
```
4. **Close it again:**
```
cryptsetup close mein_safe
```
## 3) Documentation of the Implementation
#### Flow & Functionality
Read UCI: The script checks the OpenWrt configuration. If no specific name is passed, it automatically determines all configured crypto devices.

#### Mode Switch:
##### open:
Fetches the physical device from /etc/config/cryptsetup and the target path from /etc/config/fstab. Opens the LUKS container and mounts the filesystem.
##### close:
Determines the current mount point. Checks via fuser if open files or processes are still blocking the disk. If everything is clear, it unmounts cleanly (umount) and closes the LUKS container.

#### Resources and Configuration
The script exclusively accesses the standard configurations of OpenWrt:

##### 1. Crypto Mapping: /etc/config/cryptsetup (see exampe in files)
This defines which UUID or which physical device belongs to which crypto name.
```
config cryptsetup 'backup_safe'
option device '/dev/sda1'
```
##### 2. Mount Mapping: /etc/config/fstab (see exampe in files)
This sets the mount point for the decrypted device. The reference to /dev/mapper/<name> is crucial here.
```
config mount
option target '/mnt/secure_storage'
option device '/dev/mapper/backup_safe'
option enabled '1'
```

## 4) Installation
Copy the script to your OpenWrt router, ideally to /usr/bin/ or /usr/sbin/:
```
vi /usr/bin/crypt-manage
```
Paste the script content and save the file.
Make the script executable:
```
chmod +x /usr/bin/crypt-manage
```
## 5) Examples of Invocations & Use Cases
### Usage
```
crypt-manage [ open | close ] [luks_name]* [go]
```

### Usecase 1: The Safety Test Run (Dry-Run)
By default, the script does nothing but show you what it would do. Perfect for checking your UCI configuration in advance without any risk.
```
crypt-manage open
```
Output:
```
Decrypting: backup_safe
[Dry-Run] cryptsetup open /dev/sda1 backup_safe
[Dry-Run] mount /dev/mapper/backup_safe /mnt/secure_storage
```
### Usecase 2: Live Opening and Mounting
Append the word go to the end to actually execute the command. You will be prompted interactively for your LUKS passphrase.
```
crypt-manage open go
```
### Usecase 3: Target Open of a Specific Disk
If you have multiple disks defined in the UCI but only want to open one specific disk:
```
crypt-manage open backup_safe go
```
### Usecase 4: Safe Closing (With fuser Protection)
When you want to close the disk, the script checks in the background if read or write accesses are still taking place:
```
crypt-manage close go
```
If a process (e.g., SSH or a Samba share) is still active in the directory, the script aborts and lists the culprits:
```
Locking: backup_safe
✗ Files on /mnt/secure_storage are locked/busy! Unmount aborted.
PID USER       COMMAND
1234 root       /bin/sh
```

## 6) License
This project is licensed under the MIT License – see below for details.

```
MIT License

Copyright (c) 2026 Thilo Reski

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

Developed with AI support and refined to be crisis-proof.
