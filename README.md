# Ubuntu Autoinstall — Microsoft Surface Laptop 4 (AMD)

This repository contains an unattended Ubuntu installation configuration (`user-data` + `meta-data`) tailored for a **Microsoft Surface Laptop 4 with an AMD Ryzen processor**. The autoinstall process uses Ubuntu's built-in cloud-init/curtin installer to provision the machine from scratch with zero manual input after boot.

---

## What Gets Installed

| Category | Details |
|---|---|
| **Base packages** | build-essential, curl, git, gnupg, wget, htop, jq, tmux, zsh, fzf, ripgrep, bat, neovim, stow, tree, ncdu, fd-find, unzip, zip, python3-pip, python3-venv, xdg-utils, wl-clipboard, fonts-firacode, ufw, fail2ban, dkms, apt-transport-https |
| **AMD microcode** | amd64-microcode |
| **Surface Linux kernel** | linux-image-surface, linux-headers-surface, libwacom-surface, iptsd, linux-surface-secureboot-mok, surface-control |
| **Docker** | docker-ce, docker-ce-cli, containerd.io, docker-compose-plugin |
| **MongoDB** | mongodb-org 7.0 |
| **Node.js (via NVM)** | LTS + typescript, ts-node, nodemon, pm2, yarn, @angular/cli, prettier, eslint, http-server |
| **Snaps** | VS Code (classic), Postman |
| **Shell** | Oh My Zsh with zsh-autosuggestions and zsh-syntax-highlighting |
| **Security** | UFW (SSH allowed), fail2ban enabled, LUKS full-disk encryption, SSH password auth disabled |
| **Desktop** | GNOME Tweaks, GNOME Extension Manager, Papirus icons, Arc theme |

---

## Prerequisites

- A second computer to prepare the USB drives (Linux, macOS, or Windows)
- Two USB drives:
  - **USB-A #1** — Ubuntu 22.04 LTS installer ISO (≥ 4 GB)
  - **USB-A #2** — Autoinstall config drive (any size ≥ 64 MB); or you can serve the config over HTTP
- A USB-A hub or USB-C adapter (the Surface Laptop 4 AMD has one USB-A and one USB-C port)
- Internet connection during installation (Wi-Fi is configured automatically)
- The Surface Laptop 4's UEFI Secure Boot keys must be enrolled for the Surface kernel Secure Boot MOK (handled automatically by the installer)

---

## Step 1 — Customize `user-data` Before Anything Else

Open `user-data` and update every placeholder before you put it on a USB drive. **Do not skip this.**

### 1a. Set your identity

```yaml
identity:
  hostname: surface-laptop        # change to whatever you want
  realname: Zachary Baca          # your full name
  username: zacharybaca           # your login username
  password: '$6$...'              # hashed password — see below
```

Generate a new SHA-512 hashed password on any Linux machine:

```bash
openssl passwd -6 'YourSecurePassword'
```

Paste the full `$6$...` output as the `password` value (keep the single quotes).

### 1b. Set your Wi-Fi credentials

```yaml
wifis:
  wlan0:
    access-points:
      "ItsMyWiFi":          # replace with your SSID
        password: "YourWiFiPassword"   # replace with your Wi-Fi password
```

> ⚠️ **Security:** `user-data` stores your Wi-Fi password in plaintext. **Do not commit this file to a public repository.** After installation, securely delete the `CIDATA` USB or overwrite the file.

### 1c. Set a strong LUKS disk-encryption passphrase

```yaml
storage:
  layout:
    name: direct
    encrypt: true
    password: 'CHANGE_ME_LUKS_PASSWORD'   # replace this
```

Choose a strong, memorable passphrase — you will need to type it on every boot.

### 1d. Add your SSH public key

```yaml
late-commands:
  - curtin in-target -- su - zacharybaca -c '... echo "ssh-rsa YOUR_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys ...'
```

Replace `YOUR_PUBLIC_KEY_HERE` with the full content of your `~/.ssh/id_rsa.pub` or `~/.ssh/id_ed25519.pub`. If you don't have one yet, generate it on your other machine first:

```bash
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub
```

### 1e. (Optional) Change the username throughout `user-data`

If you changed `username` in the `identity` block, do a find-and-replace for `zacharybaca` across the entire file so that paths like `/home/zacharybaca/.nvm` and `usermod` commands reference the correct user.

---

## Step 2 — Download Ubuntu 22.04 LTS

Download the **Ubuntu 22.04 LTS (Jammy Jellyfish) Desktop** ISO from the official site:

```
https://releases.ubuntu.com/22.04/ubuntu-22.04.5-desktop-amd64.iso
```

> **Why 22.04?** The `user-data` file references Jammy (`jammy`) explicitly in the Docker and MongoDB apt sources. Using a different Ubuntu release will break those repository URLs.

---

## Step 3 — Flash the Ubuntu ISO to USB #1

### On Linux / macOS

Find your USB drive device path (e.g., `/dev/sdX` or `/dev/diskN`):

```bash
# Linux
lsblk

# macOS
diskutil list
```

Write the ISO (replace `/dev/sdX` with your actual device — **this will erase the drive**):

```bash
# Linux
sudo dd if=ubuntu-22.04.5-desktop-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync

# macOS (unmount first)
diskutil unmountDisk /dev/diskN
sudo dd if=ubuntu-22.04.5-desktop-amd64.iso of=/dev/rdiskN bs=4m
```

### On Windows

Use [Rufus](https://rufus.ie/):
1. Open Rufus and select your USB drive.
2. Click **SELECT** and choose the Ubuntu ISO.
3. Partition scheme: **GPT**; Target system: **UEFI (non-CSM)**.
4. Click **START** → choose **Write in ISO Image mode** when prompted.

---

## Step 4 — Create the Autoinstall Config Drive (USB #2)

The Ubuntu installer looks for autoinstall configuration on a drive labeled **`CIDATA`** containing two files at its root: `user-data` and `meta-data`.

### On Linux

```bash
# Create a FAT32 filesystem labeled CIDATA
sudo mkfs.vfat -n CIDATA /dev/sdY   # replace sdY with your second USB drive

# Mount it
sudo mkdir -p /mnt/cidata
sudo mount /dev/sdY /mnt/cidata

# Copy the files
sudo cp user-data /mnt/cidata/user-data
sudo cp meta-data /mnt/cidata/meta-data

sudo umount /mnt/cidata
```

### On macOS

```bash
diskutil eraseDisk FAT32 CIDATA MBRFormat /dev/diskN
cp user-data /Volumes/CIDATA/user-data
cp meta-data /Volumes/CIDATA/meta-data
diskutil eject /dev/diskN
```

### On Windows

1. Format the second USB drive as FAT32 and set the label to **CIDATA** (right-click → Format in Explorer).
2. Copy `user-data` and `meta-data` to the root of the drive.

---

## Step 5 — Configure the Surface Laptop 4 UEFI

1. Power off the Surface Laptop 4 completely.
2. Hold **Volume Up** and press the **Power** button. Release both once the Surface logo appears. This boots into UEFI.
3. Make the following changes:
   - **Secure Boot** → Leave **enabled** (the Surface Linux kernel MOK enrollment handles this automatically).
   - **Boot order** → Move **USB Storage** to the top of the list.
4. Save and exit.

---

## Step 6 — Boot and Run the Installer

1. Plug both USB drives into the Surface Laptop 4. Use a USB hub if needed (one USB-A port + one USB-C port are available).
2. Power on the Surface. It should boot directly from USB #1.
3. At the GRUB menu that appears, you have two options:

   **Option A — Manually trigger autoinstall via GRUB:**
   - Highlight **"Try or Install Ubuntu"** and press **`e`** to edit the boot entry.
   - Find the line starting with `linux` and append the following to the end of that line (before `---`):
     ```
     autoinstall ds=nocloud
     ```
     The installer will scan all attached drives for one labeled `CIDATA`.
   - Press **Ctrl+X** or **F10** to boot.

   **Option B — Let the installer detect CIDATA automatically:**
   - On Ubuntu 22.04, if a drive labeled `CIDATA` containing `user-data` and `meta-data` is present at boot, the installer will detect and use it without any kernel command-line changes. Simply select **"Try or Install Ubuntu"** and press Enter.

4. The graphical installer may briefly appear, then the automated process takes over. The screen will show installation progress. **Do not touch the keyboard or mouse.**

5. When prompted (if Secure Boot MOK enrollment appears), follow the on-screen instructions to enroll the Surface kernel signing key. This typically involves:
   - Selecting **"Enroll MOK"**
   - Entering the MOK enrollment password set by the `linux-surface-secureboot-mok` package. Check the [linux-surface Secure Boot documentation](https://github.com/linux-surface/linux-surface/wiki/Secure-Boot) for the current default and instructions on changing it.
   - Rebooting to complete enrollment

6. The installer will reboot automatically when finished. **Remove both USB drives before the machine boots again**, or it may attempt to reinstall.

---

## Step 7 — First Boot

1. At the LUKS prompt, enter the disk-encryption passphrase you set in `user-data`.
2. Log in with the username and password configured in the `identity` block.
3. Open a terminal and verify the Surface kernel is running:

```bash
uname -r
# Expected output contains "surface", e.g.: 6.x.x-surface (version varies)
```

4. Verify the AMD microcode is loaded:

```bash
grep -i microcode /proc/cpuinfo | head -1
```

5. Verify Docker is running:

```bash
docker run hello-world
```

6. Verify NVM and Node.js:

```bash
nvm --version
node --version
```

7. Confirm the firewall is active:

```bash
sudo ufw status
# Should show: Status: active, with 22/tcp (OpenSSH) allowed
```

---

## Step 8 — Post-Install Recommendations

### Set your default shell to Zsh (if not already active)

The installer sets Zsh as the default shell via `usermod`. Confirm with:

```bash
echo $SHELL   # should output /usr/bin/zsh
```

If it still shows `/bin/bash`, run:

```bash
chsh -s /usr/bin/zsh
```

Log out and back in for the change to take effect.

### Configure Oh My Zsh plugins

Edit `~/.zshrc` and add `zsh-autosuggestions` and `zsh-syntax-highlighting` to the plugins list:

```bash
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
```

### Update all packages

```bash
sudo apt update && sudo apt upgrade -y
sudo snap refresh
```

### Install Surface firmware updates

The `linux-surface` repository may include firmware updates. After your first full `apt upgrade`, check:

```bash
sudo apt list --installed | grep surface
```

### Clone your dotfiles

If you manage dotfiles with GNU Stow (installed by the autoinstall), clone your dotfiles repo and run `stow` to symlink your configs.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Installer doesn't detect `CIDATA` drive | Ensure the drive is FAT32 and labeled exactly `CIDATA` (all caps). Verify both files are at the root, not in a subfolder. |
| Wi-Fi not connected during install | Double-check SSID and password in `user-data`. WPA2/WPA3 personal networks are supported; enterprise (802.1X) networks require additional configuration. |
| LUKS password prompt doesn't appear | The display driver may not be loaded. Connect an external USB keyboard and try typing the passphrase blind — the display will appear once GNOME loads. |
| Surface touch / stylus not working | Ensure `iptsd` is running: `sudo systemctl status iptsd`. The service is enabled by the installer. |
| Surface camera not working | Surface camera support is limited on Linux. Check the [linux-surface GitHub](https://github.com/linux-surface/linux-surface) for current status. |
| Docker permission denied | Log out and back in so the `docker` group membership takes effect. |
| NVM / Node not found | NVM is installed per-user in `~/.nvm`. Make sure your `~/.zshrc` or `~/.bashrc` sources NVM: `export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"` |

---

## References

- [Ubuntu Autoinstall Reference](https://ubuntu.com/server/docs/install/autoinstall-reference)
- [linux-surface Project](https://github.com/linux-surface/linux-surface)
- [Surface Laptop 4 AMD on ArchWiki](https://wiki.archlinux.org/title/Microsoft_Surface)
- [cloud-init NoCloud data source](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)
