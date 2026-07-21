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

- A second computer to prepare the installation (Linux, macOS, or Windows)
- One USB drive: Ubuntu 22.04 LTS installer ISO (≥ 4 GB)
- A USB-C to ethernet adapter — the Surface Laptop 4 AMD has no built-in ethernet port; wired ethernet is required so the installer can reach your HTTP seed source (`user-data` + `meta-data`)
- Internet connection during installation (Wi-Fi is configured automatically after the config is fetched)
- The Surface Linux kernel requires a MOK (Machine Owner Key) to be enrolled for Secure Boot. **This requires an interactive step** at the blue MOK Manager screen on first reboot — selecting "Enroll MOK" and entering the enrollment password is not automatic.

---

## Step 1 — Customize `user-data` Before Anything Else

Open `user-data` and update every placeholder before serving it. **Do not skip this.**

### 1a. Set your identity

```yaml
identity:
  hostname: surface-laptop        # change to whatever you want
  realname: Your Full Name        # your full name
  username: yourusername          # your login username
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

> ⚠️ **Security:** `user-data` stores your Wi-Fi password in plaintext and contains other sensitive values. The copy in this repository is a **sanitized template** with placeholder values — never commit a personalized copy with real credentials to a public repository. Keep your filled-in `user-data` untracked (e.g., add it to `.gitignore`) or only store it locally. Serve it only from a local HTTP server on your trusted network, or from a **private** repository / secret Gist if using GitHub (see Step 4). After installation is complete, stop the HTTP server and delete any remote copies of the filled-in file.

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
  - curtin in-target -- su - yourusername -c 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo "ssh-ed25519 YOUR_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

Replace `YOUR_PUBLIC_KEY_HERE` with the full public key string from `~/.ssh/id_ed25519.pub` (the entire line, including the leading `ssh-ed25519` type prefix). If you don't have one yet, generate it on your other machine first:

```bash
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub
```

### 1e. (Optional) Change the username throughout `user-data`

If you changed `username` in the `identity` block, do a find-and-replace for `yourusername` across the entire file so that paths like `/home/yourusername/.nvm` and `usermod` commands reference the correct user.

---

## Step 2 — Download Ubuntu 22.04 LTS

Download the **Ubuntu 22.04 LTS (Jammy Jellyfish) Desktop** ISO from the official site:

```
https://releases.ubuntu.com/22.04/ubuntu-22.04.5-desktop-amd64.iso
```

> **Why 22.04?** The `user-data` file references Jammy (`jammy`) explicitly in the Docker and MongoDB apt sources. Using a different Ubuntu release will break those repository URLs.

---

## Step 3 — Flash the Ubuntu ISO to USB

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

## Step 4 — Serve the Autoinstall Config Over HTTP

Instead of a second USB drive, the Ubuntu installer fetches `user-data` and `meta-data` from an HTTP URL you provide at the GRUB prompt. You have two options.

---

### Option A — Local HTTP server (recommended)

Run a one-liner HTTP server on your preparation machine **in the directory that contains your filled-in `user-data` and `meta-data` files**:

```bash
# Linux / macOS
python3 -m http.server 8000
```

On Windows, open PowerShell in the folder containing the files and run:

```powershell
python -m http.server 8000
```

Then find your preparation machine's local IP address:

```bash
# Linux
ip route get 1 | awk '{print $7; exit}'

# macOS
ipconfig getifaddr en0   # or en1 for Wi-Fi

# Windows (PowerShell)
(Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.IPAddress -notlike '127.*' } | Select-Object -First 1).IPAddress
# Note: if you have multiple interfaces (VPN, Docker, etc.), verify this matches your LAN adapter using `ipconfig`
```

Your seedfrom URL will be `http://<PREP_MACHINE_IP>:8000/` (trailing slash required — cloud-init appends `user-data` and `meta-data` to it; omitting the slash will cause the fetch to fail).

> **Keep the server running** until the installer has finished downloading the config. You can stop it with `Ctrl+C` once the automated install is clearly underway.

---

### Option B — GitHub Gist (hosted template)

Create a Gist that contains exactly two files named `user-data` and `meta-data` (sanitized or private if it contains real credentials). Then use this seed URL pattern:

```
http://gist.githubusercontent.com/<GIST_OWNER>/<GIST_ID>/raw/<REVISION>/
```

Cloud-init will fetch:

- `http://gist.githubusercontent.com/<GIST_OWNER>/<GIST_ID>/raw/<REVISION>/user-data`
- `http://gist.githubusercontent.com/<GIST_OWNER>/<GIST_ID>/raw/<REVISION>/meta-data`

Use the base URL with the trailing slash as your seedfrom value. GitHub will redirect the request to HTTPS automatically.

If you are using this repository's GitHub Pages-hosted files directly, the seedfrom URL is:

```
https://zacharybaca.github.io/auto-install/
```

> **Important:** Do not commit a personalized `user-data` file with real credentials to a public repository. If you host a filled-in version in a Gist, keep it private/secret and delete it after installation.

---

In either case, note your seedfrom URL — you will enter it in the GRUB boot command in Step 6.

---

## Step 5 — Configure the Surface Laptop 4 UEFI

1. Power off the Surface Laptop 4 completely.
2. Hold **Volume Up** and press the **Power** button. Release both once the Surface logo appears. This boots into UEFI.
3. Make the following changes:
   - **Secure Boot** → Leave **enabled** (you will enroll the Surface kernel MOK key on first reboot — this requires a manual interactive step at the blue MOK Manager screen).
   - **Boot order** → Move **USB Storage** to the top of the list.
4. Save and exit.

---

## Step 6 — Boot and Run the Installer

1. Plug the Ubuntu installer USB into the Surface Laptop 4. Connect the USB-C ethernet adapter and plug in a network cable so the installer can reach your seedfrom source (local HTTP server or Gist).
2. Power on the Surface. It should boot directly from the USB.
3. At the GRUB menu that appears, highlight **"Try or Install Ubuntu"** and press **`e`** to edit the boot entry.
4. Find the line starting with `linux` and append the following to the end of that line (before `---`):
   ```
   autoinstall ds=nocloud-net;seedfrom=http://192.168.1.100:8000/
   ```
   Replace `http://192.168.1.100:8000/` with your actual seedfrom URL from Step 4 (local server or Gist). The trailing `/` is required — omitting it will cause the installer to fail to locate the files.
5. Press **Ctrl+X** or **F10** to boot. The installer will fetch `user-data` and `meta-data` from the URL and begin the automated installation.

6. The graphical installer may briefly appear, then the automated process takes over. The screen will show installation progress. **Do not touch the keyboard or mouse.**

7. When prompted (if Secure Boot MOK enrollment appears), follow the on-screen instructions to enroll the Surface kernel signing key. This typically involves:
   - Selecting **"Enroll MOK"**
   - Entering the MOK enrollment password set by the `linux-surface-secureboot-mok` package. Check the [linux-surface Secure Boot documentation](https://github.com/linux-surface/linux-surface/wiki/Secure-Boot) for the current default and instructions on changing it.
   - Rebooting to complete enrollment

8. The installer will reboot automatically when finished. **Remove the USB drive before the machine boots again**, or it may attempt to reinstall.

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
| Installer doesn't fetch config (HTTP error) | Confirm your seedfrom source is reachable from the Surface. If using a local server, make sure it is still running. If using a Gist, verify the URL includes `/raw/<REVISION>/` and a trailing `/`. Also verify wired ethernet is connected and working. |
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
- [cloud-init NoCloud-Net data source](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)
