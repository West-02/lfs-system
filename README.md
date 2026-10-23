# Custom LFS (Linux From Scratch) & Minimal Package Manager (`apts`)

A personal custom Linux From Scratch (LFS) system build focused on minimalism, system transparency, and full architectural control. The core highlight of this project is a custom shell-based package management workflow named **`apts`**, designed to manage package compilation, metadata tracking, and clean installation using staging directories.

## 🛠 Architecture & Compatibility

* **Architecture:** `x86_64` (compiled on an **AMD** processor; fully compatible with Intel and other `x86_64` CPUs).

* **Userland Portability:** Thanks to standard `x86_64` instruction sets and system call conventions, compiled userland binaries run on any modern `x86_64` machine without needing to recompile the system.

* **Linux Kernel:** Configured for the target host hardware. When moving the installation to another system, you only need to reconfigure and recompile the **Linux kernel** itself — your `/usr/`, `/etc/`, and package database remain fully operational.

## 📦 How `apts` Works

`apts` is a lightweight package management script that uses staging directories to keep track of installed files.

### Core Workflow:

1. **Isolated Staging Area (`DESTDIR`):** Packages are built and installed into a temporary staging root at `/tmp/apts-$PKG_NAME` using standard build systems or manual file installation tools.

2. **Metadata Tracking:** All files placed inside the staging area are indexed into a database file at `/var/lib/apts/db/<name>-<version>.list` with absolute system paths.

3. **Target System Merge:** Files are extracted/copied from the staging folder directly into the root filesystem `/`.

4. **Source Archive Storage:** Original source archives are kept in `/var/cache/apts/` as a source of truth for versioning and database updates.

## 🚀 Realistic `apts` Build & Staging Examples

When packaging software with `apts`, standard build systems (`make`, `ninja`, `meson`) must use `DESTDIR` to direct installation outputs into the staging folder. For projects without an automated install step, `install`, `cp`, or `mv` commands are used directly targeting the staging directory.

### Example 1: Standard GNU Autotools / Make (`DESTDIR`)

```
apts "name" "version" \
    --step "./configure --prefix=/usr ..." \
    --step "make" \
    --step "make DESTDIR=/tmp/apts-"name" install"

apts libffi 3.8.0 \
    --step "./configure --prefix=/usr --disable-static --with-gcc-arch=native" \
    --step "make" \
    --step "make check" \
    --step "make DESTDIR=/tmp/apts-libffi install"
```

### Example 2: Manual Installation via `install` / `cp` / `mv`

For standalone binaries, scripts, or simple utilities that lack a `Makefile`:

```
apts "name" "version" \
    --step "install .. /tmp/apts-"name"/..." \
    --step "cp/mv "file" /tmp/apts-"name"/..."
```

## 💾 Installation & Deployment Guide

### 1. Booting via LiveUSB & Preparing Partitions

Boot any modern Linux LiveUSB (e.g., Arch Linux, Ubuntu, or LFS LiveCD) to prepare the target disk.

```
# Partition the disk (e.g., /dev/sdaX for root)
cdisk/fdisk /dev/sda

# Format partition with ext4
mkfs.ext4 /dev/sdaX

# Mount target partition
mkdir -p /mnt/lfs
mount /dev/sdaX /mnt/lfs
```

### 2. Extracting the LFS System

Extract your LFS root filesystem archive onto the mounted partition:

```
tar -xvf lfs-system.tar.xz -C /mnt/lfs
```

### 3. Generating `/etc/fstab`

Obtain the UUID of your root partition and write `/etc/fstab`:

```
# Find root partition UUID
blkid /dev/sda1

# Create /mnt/lfs/etc/fstab
cat << EOF > /mnt/lfs/etc/fstab
# /etc/fstab: Static file system information
# <file system>                           <mount point>   <type>  <options>       <dump>  <pass>
UUID=XXXX-XXXX-XXXX-XXXX                  /               ext4    defaults        1       1
EOF
```

### 4. Setting up GRUB Bootloader

If GRUB is already installed on the system, add an entry for your new LFS kernel to `/boot/grub/grub.cfg`.

#### Chroot into LFS environment:

```
mount --bind /dev /mnt/lfs/dev
mount --bind /proc /mnt/lfs/proc
mount --bind /sys /mnt/lfs/sys

chroot /mnt/lfs /bin/bash
```

#### Install GRUB to MBR/EFI (if GRUB is not present):

```
# For legacy MBR
grub-install --target=i386-pc --recheck /dev/sdX

# For EFI
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=LFS


# Generate configuration
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Custom GRUB Entry (if integrating into an existing GRUB setup):

You need `os-prober` for GRUB to detect new Linux kernels on other partitions

```
# Generate configuration on system
sudo grub-mkconfig -o /boot/grub/grub.cfg

vim/nano /boot/grub/grub.cfg

menuentry "Custom Linux From Scratch" {
    linux /boot/vmlinuz-linux root=UUID=XXXX-XXXX-XXXX-XXXX(or partition) rw quiet init=/lib/systemd/systemd
}
```
