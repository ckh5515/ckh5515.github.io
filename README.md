# Arch Linux Installation
Charles Harvey  
CYB-3353 System Administration

# 1. Pre-installation

I installed VMware Workstation Pro 26H1. I created a VM with 4 GB of RAM, 2 processor cores, and a 40 GB virtual disk.

I used NAT networking and UEFI firmware with Secure Boot disabled.

## 1.1 Acquire an installation image

I downloaded `archlinux-2026.10.01-x86_64.iso` from the MIT mirror:

https://mirrors.mit.edu/archlinux/iso/2026.10.01/

I also downloaded the signature file and `sha256sums.txt`.

## 1.2 Verify signature

I checked whether GnuPG was installed in PowerShell:

```powershell
gpg --version
```

The command was not available, so I did not complete PGP signature verification.

I checked the ISO's SHA256 checksum instead:

```powershell
Get-FileHash -Algorithm SHA256 "C:\Users\charl\OneDrive\Desktop\TU\Fall 2026\CYB-3353 Sys Admin\arch-install\archlinux-2026.10.01-x86_64.iso"
```

The result matched the hash in `sha256sums.txt`. This checked the downloaded file against the published checksum.

## 1.3 Prepare an installation medium

I attached the ISO to the VM's virtual CD/DVD drive.

## 1.4 Boot the live environment

I started the VM and booted from the ISO. The live environment opened a terminal logged in as root.

## 1.5 Set the console keyboard layout and font

I kept the default US keyboard layout and console font.

## 1.6 Verify the boot mode

I ran this command to check the UEFI platform size:

```bash
cat /sys/firmware/efi/fw_platform_size
```

The output was `64`, confirming 64-bit UEFI mode.

## 1.7 Connect to the internet

I tested the connection:

```bash
ping archlinux.org
```

I received replies and pressed Ctrl + C to stop the test. There was no packet loss.

## 1.8 Update the system clock

I checked the clock:

```bash
timedatectl
```

The output showed that the clock was synchronized and NTP was active.

## 1.9 Partition the disks

I listed the disks:

```bash
lsblk
```

The VM's 40 GB disk was `/dev/sda`.

I opened the partition editor:

```bash
cfdisk /dev/sda
```

I selected GPT and created the partitions below.

I initially exited without writing the changes. I reopened the editor, recreated the partitions, selected Write, confirmed with `yes`, and quit.

### 1.9.1 Example layouts

I used this layout:

- `/dev/sda1`: 1 GB EFI System Partition, mounted at `/boot`
- `/dev/sda2`: 4 GB swap partition
- `/dev/sda3`: remaining space for the root filesystem

I checked that the partitions were saved:

```bash
lsblk /dev/sda
```

## 1.10 Format the partitions

I formatted the EFI partition as FAT32:

```bash
mkfs.fat -F 32 /dev/sda1
```

I prepared the swap partition:

```bash
mkswap /dev/sda2
```

I formatted the root partition as ext4:

```bash
mkfs.ext4 /dev/sda3
```

I checked the filesystem types:

```bash
lsblk -f /dev/sda
```

## 1.11 Mount the file systems

I mounted the root partition:

```bash
mount /dev/sda3 /mnt
```

I created the boot folder and mounted the EFI partition:

```bash
mount --mkdir /dev/sda1 /mnt/boot
```

I enabled swap:

```bash
swapon /dev/sda2
```

# 2. Installation

## 2.1 Select the mirrors

I used the mirror list provided by the live environment.

## 2.2 Install essential packages

I installed the base system, Linux kernel, firmware, and Nano editor:

```bash
pacstrap -K /mnt base linux linux-firmware nano
```

The installation completed successfully.

# 3. Configure the system

## 3.1 Fstab

I generated the filesystem mount configuration using UUIDs:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

I checked the file:

```bash
cat /mnt/etc/fstab
```

It listed the root, boot, and swap partitions.

## 3.2 Chroot

I entered the installed system:

```bash
arch-chroot /mnt
```

This allowed me to configure the new installation.

## 3.3 Time

I set the timezone to America/Chicago:

```bash
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```

I updated the hardware clock from the system clock:

```bash
hwclock --systohc
```

I checked the time:

```bash
date
```

## 3.4 Localization

I enabled US English UTF-8 in the locale configuration:

```bash
sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
```

I generated the locale:

```bash
locale-gen
```

I set the default language:

```bash
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

I saved the US console keyboard layout:

```bash
echo KEYMAP=us > /etc/vconsole.conf
```

## 3.5 Network configuration

I set the hostname to `archlinux`:

```bash
echo archlinux > /etc/hostname
```

I checked the language, keyboard, and hostname settings:

```bash
cat /etc/locale.conf /etc/vconsole.conf /etc/hostname
```

I installed NetworkManager and the GRUB packages:

```bash
pacman -S networkmanager grub efibootmgr
```

I enabled NetworkManager to start when the system boots:

```bash
systemctl enable NetworkManager
```

## 3.6 Initramfs

The initial installation warned that `/etc/vconsole.conf` was missing.

After creating that file, I rebuilt the initramfs:

```bash
mkinitcpio -P
```

The initramfs contains files needed during startup. The rebuild completed successfully.

## 3.7 Root password

I set the root password:

```bash
passwd
```

I entered the password twice.

## 3.8 Boot loader

I installed GRUB for UEFI:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

I generated its configuration file:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

GRUB found the Linux kernel and initramfs.

The command showed warnings about detecting other operating systems. This VM only contained Arch Linux.

# 4. Reboot

I exited the chroot:

```bash
exit
```

I unmounted the filesystems:

```bash
umount -R /mnt
```

I powered off the VM:

```bash
poweroff
```

In VMware settings, I unchecked Connect at power on for the CD/DVD drive.

I started the VM again and logged into the installed system as root.

I tested the connection:

```bash
ping -c 3 archlinux.org
```

All three packets received replies.

# 5. Post-installation

## 5.1 Install sudo, Zsh, and SSH

I installed the required packages:

```bash
pacman -S sudo zsh openssh
```

Sudo allows administrator commands. Zsh provides a shell other than Bash. OpenSSH provides remote access.

## 5.2 Create user accounts

I created my account with a home directory, wheel group membership, and Zsh as its shell:

```bash
useradd -m -G wheel -s /bin/zsh charles
```

I set my password:

```bash
passwd charles
```

I created the codi account:

```bash
useradd -m -G wheel -s /bin/zsh codi
```

I set its password:

```bash
passwd codi
```

## 5.3 Configure sudo permissions

I created a rule allowing wheel group members to use sudo:

```bash
echo '%wheel ALL=(ALL:ALL) ALL' > /etc/sudoers.d/10-wheel
```

I set the file permissions:

```bash
chmod 440 /etc/sudoers.d/10-wheel
```

I checked the configuration for errors:

```bash
visudo -c
```

The files passed the check.

I checked both users' group memberships:

```bash
id charles
id codi
```

Both accounts belonged to the wheel group.

## 5.4 Enable SSH

I started SSH and enabled it at boot:

```bash
systemctl enable --now sshd
```

I checked the service:

```bash
systemctl is-active sshd
```

The output was `active`.

## 5.5 Install the desktop environment

I updated the system and installed the desktop packages:

```bash
pacman -Syu --noconfirm xorg-server lxqt openbox sddm breeze-icons ttf-dejavu firefox open-vm-tools
```

I used LXQt for the desktop, Openbox for the window manager, and SDDM for the graphical login screen.

The command also installed icons, fonts, Firefox, and VMware tools.

Some package names were initially misspelled, which caused “target not found” errors. I corrected the names and reran the installation.

I enabled VMware tools:

```bash
systemctl enable --now vmtoolsd
```

I enabled SDDM to start at boot:

```bash
systemctl enable sddm
```

I restarted the VM:

```bash
reboot
```

The graphical login screen appeared. I selected LXQt Desktop and logged in as charles.

## 5.6 Verify the user account and shell

I opened a terminal and checked my username:

```bash
whoami
```

The output was `charles`.

I checked my shell:

```bash
echo $SHELL
```

The output was `/bin/zsh`.

I tested sudo access:

```bash
sudo whoami
```

After entering my password, the output was `root`.

## 5.7 Add terminal colors and aliases

I opened my Zsh configuration:

```bash
nano ~/.zshrc
```

I added these lines:

```zsh
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f %# '
alias ls='ls --color=auto'
alias ll='ls -lah'
alias update='sudo pacman -Syu'
```

The prompt uses green for the username and hostname and blue for the current directory.

The aliases provide colored file listings, detailed file listings, and a shortcut for system updates.

I saved the file and loaded the changes:

```bash
source ~/.zshrc
```

I tested the detailed file listing:

```bash
ll
```

I displayed the aliases:

```bash
alias
```

The colored prompt and aliases worked.

## 5.8 Install an AUR helper

I installed the tools needed to build packages:

```bash
sudo pacman -S --needed git base-devel
```

I moved to my home directory:

```bash
cd ~
```

I downloaded the yay-bin build files from the AUR:

```bash
git clone https://aur.archlinux.org/yay-bin.git
```

I entered the folder:

```bash
cd yay-bin
```

I displayed the package build instructions:

```bash
cat PKGBUILD
```

I built and installed the package:

```bash
makepkg -si
```

I checked that Yay was installed:

```bash
yay --version
```

I also checked the installed package version:

```bash
pacman -Q yay-bin
```

The installed package was `yay-bin 13.1.0-1`.

# References

Arch Linux Installation Guide:

https://wiki.archlinux.org/title/Installation_guide

Arch Linux ISO mirror:

https://mirrors.mit.edu/archlinux/iso/2026.10.01/

Yay:

https://github.com/Jguer/yay
