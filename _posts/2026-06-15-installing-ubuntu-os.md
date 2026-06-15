---
title:  "Installing Ubuntu OS"
date:   2026-06-15 11:16:55 +0530
categories:
  - blog
tags:
  - Ubuntu
  - OS
---

## Reinstalling my OS 

A short, no-drama checklist for when I reinstall Ubuntu (encrypted, clean install).
The whole game is: **back up the right stuff off the machine, wipe the right disk,
then put everything back where it belongs.** That's it.

The backup folder I use is `~/backup-info-new`. Copy that whole folder onto a pendrive / external drive before doing anything destructive.

---

## 1. Before I wipe anything (pre-boot)

### What goes in the backup folder

These are the things that actually hurt to lose. Everything else I can reinstall.

- `.ssh` - SSH keys (the most valuable thing here)
- `.gitconfig` - git identity + settings
- `.gnupg` - GPG keys
- `wireguard` - my VPN `.conf` files (these live in `/etc/wireguard` on the system)
- `.aws`, `.azure` - cloud CLI credentials
- `vscode-essential` - settings.json, keybindings.json, snippets
- `.zshrc` + `.oh-my-zsh` - shell setup
- package lists - so I remember what I had installed

Rough idea of building the folder:

```bash
mkdir -p ~/backup-info-new

cp -r ~/.ssh        ~/backup-info-new/
cp    ~/.gitconfig  ~/backup-info-new/
cp -r ~/.gnupg      ~/backup-info-new/
cp -r ~/.aws        ~/backup-info-new/
cp -r ~/.azure      ~/backup-info-new/
cp    ~/.zshrc      ~/backup-info-new/
cp -r ~/.oh-my-zsh  ~/backup-info-new/

# WireGuard lives under /etc, so it needs sudo
sudo cp -r /etc/wireguard ~/backup-info-new/wireguard
sudo chown -R $USER:$USER ~/backup-info-new/wireguard

# VS Code settings
mkdir -p ~/backup-info-new/vscode-essential
cp ~/.config/Code/User/settings.json    ~/backup-info-new/vscode-essential/ 2>/dev/null
cp ~/.config/Code/User/keybindings.json ~/backup-info-new/vscode-essential/ 2>/dev/null
cp -r ~/.config/Code/User/snippets      ~/backup-info-new/vscode-essential/ 2>/dev/null

# Lists, not the actual packages
code --list-extensions > ~/backup-info-new/vscode-extensions.txt
apt-mark showmanual    > ~/backup-info-new/manually-installed.txt
```

Then copy the whole folder onto the pendrive:

```bash
cp -r ~/backup-info-new /media/$USER/<DRIVE_NAME>/
```

### Verify it actually made it onto the drive

This is the step I'm tempted to skip and shouldn't. The backup has to exist
**off the machine**, not just in my home folder.

```bash
ls /media/$USER                                        # find the drive name
ls -la /media/$USER/<DRIVE_NAME>/backup-info-new       # is the folder there?
```

I want to see `.ssh`, `.gnupg`, `.aws`, `.azure`, `wireguard`, `vscode-essential`,
and the package lists.

Spot-check the two that matter most:

```bash
# SSH keys present?
find /media/$USER/<DRIVE_NAME>/backup-info-new/.ssh -type f

# WireGuard configs present?
ls -la /media/$USER/<DRIVE_NAME>/backup-info-new/wireguard   # should show .conf files
```

Sanity check the sizes roughly match the original:

```bash
du -sh /media/$USER/<DRIVE_NAME>/backup-info-new
du -sh ~/backup-info-new
```

If `.ssh` is there and the sizes are close, I'm safe to proceed.

---

## 2. Get the ISO

Download the Ubuntu ISO from the official site. [link](https://ubuntu.com/download/desktop)

---

## 3. Make the pendrive bootable (GNOME Disks)

Checkout this quick youtube video, which I refered when I booted my pendrive for the first time. [link](https://youtu.be/PurlSJCCuQQ?si=Z0VyEb7INhsQX67v)

> Note: the install pendrive (with the ISO) and the backup drive should ideally be
> two **different** drives. If I'm reusing a stick that already has an old bootable
> image on it, writing the new image over it cleans it - that's fine.

Using the **Disks** utility:

1. Open **Disks**.
2. Select the USB stick in the left panel (double-check the size so it's the right one).
3. Hamburger menu (top-right) → **Restore Disk Image…**
4. Choose the downloaded `.iso`.
5. **Start Restoring** → confirm. This wipes the stick and writes the bootable image.

---

## 4. Boot from it

I refered this video for installing the ubuntu: [link](https://youtu.be/zt0ZNYRBN1M?si=qgweklye98voNtVh)

1. Reboot, and at startup spam the boot-menu / BIOS key (F12 / F2 / Esc / Del,
   depending on the machine).
2. Pick the USB stick as the boot device.
3. Start the Ubuntu installer.

---

## 5. After install - put everything back

Plug the pendrive into the fresh OS and copy `backup-info-new` into the new home
folder. Then restore in this order (the order matters least, but this is what gets
me productive fastest):

### 1. SSH keys
```bash
cp -r ~/backup-info-new/.ssh ~/
chmod 700 ~/.ssh
chmod 600 ~/.ssh/*
ls -la ~/.ssh
```

### 2. Git config
```bash
cp ~/backup-info-new/.gitconfig ~/
git config --list
```

### 3. GPG keys
```bash
cp -r ~/backup-info-new/.gnupg ~/
chmod 700 ~/.gnupg
gpg --list-keys
```

### 4. WireGuard
```bash
sudo apt update
sudo apt install wireguard wireguard-tools
sudo cp -r ~/backup-info-new/wireguard/* /etc/wireguard/
sudo chmod 600 /etc/wireguard/*.conf
sudo wg-quick up wg0        # swap wg0 for whatever my config is named
```

### 5. Zsh + Oh My Zsh
```bash
sudo apt install zsh
cp ~/backup-info-new/.zshrc ~/
cp -r ~/backup-info-new/.oh-my-zsh ~/
chsh -s $(which zsh)        # takes effect after log out / back in
```

### 6. VS Code
```bash
sudo snap install code --classic

mkdir -p ~/.config/Code/User
cp ~/backup-info-new/vscode-essential/settings.json    ~/.config/Code/User/ 2>/dev/null
cp ~/backup-info-new/vscode-essential/keybindings.json ~/.config/Code/User/ 2>/dev/null
cp -r ~/backup-info-new/vscode-essential/snippets      ~/.config/Code/User/ 2>/dev/null

# extensions
cat ~/backup-info-new/vscode-extensions.txt | xargs -L 1 code --install-extension
```

### 7. AWS credentials
```bash
cp -r ~/backup-info-new/.aws ~/
aws sts get-caller-identity   # if AWS CLI is installed
```

### 8. Azure credentials
```bash
cp -r ~/backup-info-new/.azure ~/
az account show               # if Azure CLI is installed
```

### 9. Reinstall packages (selectively!)
```bash
cat ~/backup-info-new/manually-installed.txt
```
Don't blindly reinstall 1000+ packages. Just pull back the ones I actually use.
Usual suspects: Docker, kubectl, Helm, Google Chrome, VS Code, Flameshot, Albert,
pgAdmin 4, Go, Homebrew (if I still want it).

---
## Additional things I installed are

First I installed the docker, git, vscode

Get the gnome-extension-manager
```bash
sudo apt install gnome-shell-extension-manager
```
In gnome extension manager get the following extensions
`Clipboard Indicator` by Tudmotu.com 
`Touchpad Gesture Customization` by coooolapps.com

Also install the flameshot for capturing interactive screenshots.

---

## The very first thing to check after restoring SSH

```bash
ssh -T git@github.com
```

If that succeeds, the most valuable part of my setup is back and I'm basically
operational. Everything else is just convenience from here.