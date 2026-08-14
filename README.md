# Fedora + Btrfs + KDE Plasma + GRUB + NVIDIA (bootstrap install)

Personal notes for building Fedora the "Arch way" — every decision made by hand, nothing you don't ask for. Btrfs `@` subvolumes, KDE Plasma, GRUB, Snapper, NVIDIA, and the full codec/non-free repos. Mirror of my [Arch-Linux-Install-Memo](https://github.com/pop/Arch-Linux-Install-Memo).

Not the best or most correct way. Just the way I like.

> ✅ **Every package name below was verified against live Fedora 44, RPM Fusion, Terra, and COPR repo metadata.** See [Package Map](#package-map-arch--fedora).

## Contents
1. [Assumptions](#assumptions)
2. [Partition & Format](#partition--format)
3. [Subvolumes & Mounts](#subvolumes--mounts)
4. [Bootstrap Base](#bootstrap-base)
5. [Chroot Config](#chroot-config)
6. [Repositories](#repositories)
7. [GRUB Bootloader](#grub-bootloader)
8. [Services](#services)
9. [Desktop Stack](#desktop-stack)
10. [Codecs & Non-Free](#codecs--non-free)
11. [Snapper & Btrfs Tools](#snapper--btrfs-tools)
12. [NVIDIA Driver](#nvidia-driver)
13. [SELinux](#selinux)
14. [Reboot](#reboot)
15. [Post-Install](#post-install)
16. [Package Map (Arch → Fedora)](#package-map-arch--fedora)
17. [Credits](#credits)

---

## Assumptions
- UEFI, Secure Boot off, single NVMe `/dev/nvme0n1`.
- ESP partition first, swap last. User `pop`, hostname `fedora`, TZ `Asia/Bangkok`.
- Deviate? Adjust disk paths, UUIDs, and timezone.

### Which ISO?
Any Fedora **44** Live ISO works — the bootstrap only needs a live shell, so the desktop it boots doesn't matter. Match the release exactly: the `--releasever=44` in this guide and the `-44` in the RPM Fusion/Terra URLs must equal the ISO release.

- **Workstation Live** — GNOME desktop in the live session; easiest if you want a GUI while installing. Good default.
- **KDE Plasma Spin (Live)** — if you want the live session to look like the target system.
- **Server (bare-metal) ISO** — headless/no-GUI target; boots into the installer, but you can drop to a root shell on a TTY (`Ctrl+Alt+F2`) and follow this guide from there.

Download from https://getfedora.org/ and verify the checksum before writing to USB (Fedora publishes `.CHECKSUM` files next to each ISO).

---

## Partition & Format

| Partition | Size | Type | Purpose |
|-----------|------|------|---------|
| `/dev/nvme0n1p1` | 2-4 GB | EFI System | `/boot` |
| `/dev/nvme0n1p2` | Remainder | Linux filesystem | Btrfs root |
| `/dev/nvme0n1p3` | ~RAM | swap | swap |

```bash
cfdisk /dev/nvme0n1                          # GPT
mkfs.btrfs -f -L FedoraFS /dev/nvme0n1p2
mkswap /dev/nvme0n1p3

esp_uuid="$(blkid -s UUID -o value /dev/nvme0n1p1)"
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"
```

---

## Subvolumes & Mounts
Same `@` layout as my Arch memo so Snapper semantics carry over.

```bash
mount UUID="${root_uuid}" /mnt
for s in @ @home @var @var_log @var_cache @root @srv; do
  btrfs subvolume create /mnt/$s
done
umount -R /mnt

mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home      UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var       UUID="${root_uuid}" /mnt/var
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_log   UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@root      UUID="${root_uuid}" /mnt/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@srv       UUID="${root_uuid}" /mnt/srv

mount --mkdir UUID="${esp_uuid}" /mnt/boot
swapon UUID="${swap_uuid}"

mkdir -p /mnt/etc
genfstab -U /mnt > /mnt/etc/fstab           # install arch-install-scripts on live ISO
cat /mnt/etc/fstab
```

---

## Bootstrap Base
`dnf --installroot` is Fedora's `pacstrap`. First write a repo file so dnf knows where to pull from.

```bash
mkdir -p /mnt/etc/yum.repos.d
cat > /mnt/etc/yum.repos.d/fedora.repo <<'EOF'
[fedora]
name=Fedora $releasever - $basearch
metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-$releasever&arch=$basearch
enabled=1
countme=1
metadata_expire=7d
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch
skip_if_unavailable=False
EOF

dnf --installroot=/mnt --releasever=44 group install "Core"

for f in dev proc sys sys/firmware/efi/efivars; do mount --bind /$f /mnt/$f; done
cp /etc/resolv.conf /mnt/etc/resolv.conf
chroot /mnt /bin/bash
```

---

## Chroot Config
```bash
dnf group install "Standard" "Development Tools"    # ≈ base-devel
dnf install kernel kernel-core linux-firmware microcode_ctl \
  btrfs-progs dosfstools e2fsprogs exfatprogs efibootmgr \
  NetworkManager openssh-server \
  vim-enhanced neovim git sudo man-db curl perl \
  zsh zsh-autosuggestions bash-completion \
  pipewire pipewire-alsa pipewire-pulseaudio pipewire-jack-audio-connection-kit wireplumber \
  inotify-tools dracut \
  grub2-efi-x64 grub2-tools shim-x64 os-prober \
  langpacks-en 7zip netcat
```

```bash
ln -sf /usr/share/zoneinfo/Asia/Bangkok /etc/localtime
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
echo 'fedora' > /etc/hostname
cat > /etc/hosts <<'EOF'
127.0.0.1   localhost
::1         localhost
127.0.1.1   fedora.localdomain fedora
EOF

passwd
useradd -mG wheel,storage,power,audio,video -s /bin/bash pop
passwd pop
chown -R pop:pop /home/pop
visudo                                          # uncomment: %wheel ALL=(ALL) ALL

dracut --regenerate-all --force                 # btrfs + microcode auto-included
```

---

## Repositories
Add RPM Fusion, Terra, and the COPR plugin. Anything not here is available from one of these or Flatpak.

```bash
# RPM Fusion (free + nonfree + tainted)
dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-44.noarch.rpm \
            https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-44.noarch.rpm

# Terra (Fedora's AUR-ish community repo)
dnf install --nogpgcheck --repofrompath 'terra,https://repos.fyralabs.com/terra$releasever' \
            terra-release terra-gpg-keys
dnf install terra-release-extras terra-release-mesa    # optional subrepos

# COPR plugin
dnf install 'dnf-command(copr)'
```

> Terra `nvidia`/`multimedia` subrepos conflict with RPM Fusion packages — pick one set. RPM Fusion is used below.

---

## GRUB Bootloader
Fedora-native; kernel updates auto-run `kernel-install` → `dracut` + `grub2-mkconfig`. No manual copying.

```bash
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"
cat > /etc/default/grub <<EOF
GRUB_TIMEOUT=3
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rhgb quiet loglevel=3 rootflags=subvol=@ resume=UUID=${swap_uuid} zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=50 zswap.zpool=zsmalloc"
GRUB_DISABLE_RECOVERY="true"
EOF

grub2-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Fedora
grub2-mkconfig -o /boot/grub2/grub.cfg

# dual-boot? uncomment for os-prober:
# echo 'GRUB_DISABLE_OS_PROBER=false' >> /etc/default/grub
# grub2-mkconfig -o /boot/grub2/grub.cfg
```

---

## Services
```bash
dnf install util-linux usbutils rsync htop bat zip unzip \
  iwd avahi nss-mdns alsa-utils alsa-sof-firmware easyeffects \
  bluez cups acpi acpid xdg-user-dirs

systemctl enable NetworkManager bluetooth sshd fstrim.timer
# systemctl enable cups avahi-daemon acpid iwd
```

---

## Desktop Stack
```bash
dnf install sddm sddm-kcm \
  xdg-desktop-portal xdg-desktop-portal-kde qt6-qtwayland xorg-x11-server-Xwayland \
  plasma-desktop plasma-workspace kwin plasma-systemsettings \
  plasma-nm plasma-pa kscreen kde-gtk-config breeze-gtk \
  bluedevil power-profiles-daemon kdeplasma-addons plasma-systemmonitor \
  plasma-browser-integration plasma-discover krdp plasma-print-manager \
  appmenu-qt5 libdbusmenu kdeconnectd plasma-vault \
  dolphin dolphin-plugins konsole kate okular gwenview spectacle ark \
  gparted kio-extras ffmpegthumbs kdegraphics-thumbnailers filelight kcalc

systemctl enable sddm power-profiles-daemon
```

---

## Codecs & Non-Free
```bash
# full multimedia codecs (replaces ffmpeg-free)
dnf install ffmpeg ffmpeg-libs libavcodec-freeworld \
  gstreamer1-plugins-ugly gstreamer1-plugins-bad-freeworld \
  x264 x265 gstreamer1-plugin-openh264 mesa-va-drivers-freeworld

# DVD CSS
dnf install --enablerepo=rpmfusion-free-tainted libdvdcss

# players + thumbnails (Fedora)
dnf install vlc mpv ffmpegthumbnailer

# non-free: Steam, MS core fonts (license-fetched on install)
dnf install steam lpf-mscore-fonts
```

---

## Snapper & Btrfs Tools
```bash
dnf install snapper btrfs-assistant btrfsmaintenance
snapper -c root create-config /
snapper -c home create-config /home

# snap-pac equivalent: auto-snapshot on every dnf transaction
dnf copr enable biyuan/dnf5-autosnapper
dnf install dnf5-autosnapper
```

---

## NVIDIA Driver
```bash
# proprietary driver
dnf install akmod-nvidia xorg-x11-drv-nvidia \
  xorg-x11-drv-nvidia-cuda nvidia-settings xorg-x11-drv-nvidia-libs

# ...or the open kernel module (matches Arch's nvidia-open-dkms)
# dnf install --enablerepo=rpmfusion-nonfree-tainted akmod-nvidia-open
```
```bash
echo 'nvidia-drm.modeset=1 nvidia-drm.fbdev=1' >> /etc/default/grub   # add to GRUB_CMDLINE_LINUX
grub2-mkconfig -o /boot/grub2/grub.cfg
# first boot rebuilds the akmod DKMS module; reboot once more
```

---

## SELinux
Your Arch box has none — pick one:
- **Keep enforcing** (recommended): `touch /.autorelabel`, boot once with `selinux=0` (press `e` in GRUB, append to cmdline), relabel runs, reboot to enforcing.
- **Disable** (feels like Arch): `sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config` (skip `/.autorelabel`).

---

## Reboot
```bash
exit
umount -R /mnt
swapoff -a
reboot
```

---

## Post-Install
```bash
# Nerd Fonts (COPR) — pick the faces you want
dnf copr enable komapro/nerd-fonts
dnf install jetbrainsmono-nerd-font firacode-nerd-font

# fonts
sudo dnf install google-noto-sans-fonts google-noto-serif-fonts \
  google-noto-sans-cjk-fonts google-noto-emoji-fonts \
  dejavu-sans-fonts liberation-fonts-all \
  jetbrains-mono-fonts fira-code-fonts terminus-fonts \
  adobe-source-sans-pro-fonts adobe-source-serif-pro-fonts adobe-source-code-pro-fonts

# flatpak
sudo dnf install flatpak
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

```bash
# apps (Fedora) — pick what you need
sudo dnf install firefox chromium libreoffice filezilla \
  gimp obs-studio gamemode mangohud lutris goverlay \
  ImageMagick gvfs gvfs-smb brightnessctl clinfo libva-utils mpv
```

```bash
# optional: CachyOS kernels (COPR port, like the CachyOS section of my Arch memo)
# dnf copr enable crono/kernel-cachyos
# dnf install kernel-cachyos kernel-cachyos-devel
```

```bash
# unofficial/streaming apps → Flatpak (mailspring, vscode, brave, protonup)
# flatpak install -y flathub com.mailspring.Mailspring com.visualstudio.code
```

---

## Package Map (Arch → Fedora)
All Fedora entries below were verified to exist in the Fedora 44 repos; RPM Fusion / Terra / COPR entries verified against their live repodata.

| Arch (your memo) | Fedora (verified) |
|---|---|
| `base`, `base-devel` | group `Core`, `Standard`, `Development Tools` |
| `linux`, `linux-headers`, `linux-firmware` | `kernel`, `kernel-core`, `linux-firmware` |
| `intel-ucode` / `amd-ucode` | `microcode_ctl` (embedded by dracut) |
| `efibootmgr` | `efibootmgr` |
| `btrfs-progs` | `btrfs-progs` |
| `openssh` | `openssh-server` |
| `vim`, `nvim`, `git`, `sudo`, `man` | `vim-enhanced`, `neovim`, `git`, `sudo`, `man-db` |
| `zsh`, `zsh-completions`, `bash-completion` | `zsh`, (zsh ships compinit), `bash-completion` |
| `pipewire`, `-alsa`, `-pulse`, `-jack`, `wireplumber` | `pipewire`, `pipewire-alsa`, `pipewire-pulseaudio`, `pipewire-jack-audio-connection-kit`, `wireplumber` |
| `reflector` | n/a (Fedora metalinks) |
| `inotify-tools` | `inotify-tools` |
| `mkinitcpio` | `dracut` |
| Limine | `grub2-efi-x64 grub2-tools shim-x64` + `grub2-mkconfig` |
| `sof-firmware` | `alsa-sof-firmware` |
| `bluez-utils` | `bluez` (merged) |
| `p7zip` | `7zip` |
| `openbsd-netcat` | `netcat` |
| `imagemagick` | `ImageMagick` |
| `systemsettings` | `plasma-systemsettings` |
| `discover` | `plasma-discover` |
| `print-manager` | `plasma-print-manager` |
| `appmenu-gtk-module`, `libdbusmenu-glib` | `appmenu-qt5`, `libdbusmenu` |
| `kdeconnect` | `kdeconnectd` |
| `qt6-wayland`, `xorg-xwayland` | `qt6-qtwayland`, `xorg-x11-server-Xwayland` |
| `gamemode`, `mangohud`, `goverlay`, `lutris` | same (Fedora official) |
| `steam` (non-free) | `steam` (RPM Fusion nonfree) |
| `ffmpeg`, codecs | `ffmpeg ffmpeg-libs libavcodec-freeworld gstreamer1-plugins-ugly gstreamer1-plugins-bad-freeworld x264 x265` (RPM Fusion free) |
| `libdvdcss` | `libdvdcss` (RPM Fusion free-tainted) |
| `nvidia-open-dkms` | `akmod-nvidia-open` (RPM Fusion nonfree-tainted) or `akmod-nvidia` |
| `cuda`, `opencl-nvidia` | not in RPM Fusion F44; use NVIDIA's CUDA repo / driver-built OpenCL |
| `yay` / AUR | `dnf copr enable` + Terra + RPM Fusion + Flatpak |
| `snap-pac` | `dnf5-autosnapper` (COPR biyuan/dnf5-autosnapper) |
| `limine-snapper-sync` | n/a — Snapper is your choice; GRUB boots snapshots directly |
| `btrfs-assistant`, `snapper-gui` | `btrfs-assistant` (Fedora), `snapper` |
| `ttf-*` fonts | `google-noto-*-fonts`, `dejavu-sans-fonts`, `jetbrains-mono-fonts`, `fira-code-fonts`, `liberation-fonts-all`, `adobe-source-*-pro-fonts`, `terminus-fonts` |
| `nerd-fonts` | COPR `komapro/nerd-fonts` (e.g. `jetbrainsmono-nerd-font`) |
| `ttf-ms-fonts` | `lpf-mscore-fonts` (RPM Fusion nonfree) |
| `cachyos` kernels | COPR `crono/kernel-cachyos` (optional) |

**Dropped for leanness** (no clean Fedora equivalent or redundant): `reflector`, `zsh-completions`, `wget` (curl covers), `mailspring-bin`/`brave-bin`/`visual-studio-code-bin`/`proton-ge-custom-bin` (Flatpak instead), `fatfetch`, `lib32-gamemode`/`lib32-mangohud` (no F44 RPM Fusion 32-bit), `ubuntu-font-family`, `noto-fonts-extra`, `terminus-font`→`terminus-fonts`.

---

## Credits
- Fedora bootstrap: https://blog.mtaha.dev/linux/fedora_bootstrap
- Btrfs + Snapper layout: https://www.ordinatechnic.com/distribution-specific-guides/fedora/a-fedora-installation-on-a-btrfs-filesystem-with-a-simple-snapper-compatible-subvolume-layout
- Fedora bootstrap notes: https://github.com/samwhelp/fedora-bootstap-notes
- Terra: https://docs.terrapkg.com/usage/installing
- RPM Fusion config: https://rpmfusion.org/Configuration
