---
name: arch_install_guide
description: Full Arch Linux installation guide for laptop with NVIDIA 3070 — Hyprland, Btrfs, systemd-boot, Zsh+Starship, AI/ML stack
type: project
---

## Arch Linux Install — Laptop with NVIDIA 3070

**Choices**: Hyprland (Wayland), Btrfs (subvolumes @, @home, @snapshots, @var_log), systemd-boot, Zsh + Starship
**Apps**: Claude Code, Firefox, Spotify, Discord, Alacritty
**RAM**: 32GB, Swap: 32GB

**Why:** User is setting up a fresh laptop optimized for AI/dev work.
**How to apply:** Reference this when helping with post-install config, troubleshooting, or dotfiles.

### Install Phases
1. **Pre-install** — iwctl, partition (512MB EFI + 32GB swap + rest btrfs), btrfs subvolumes, pacstrap
2. **Base config** — timezone, locale, hostname, user `noel`, systemd-boot with `nvidia_drm.modeset=1`
3. **NVIDIA** — nvidia + nvidia-utils + cuda + cudnn, early-load modules in mkinitcpio, pacman hook for auto-rebuild
4. **Hyprland** — hyprland + waybar + wofi + dunst + pipewire + bluetooth, NVIDIA env vars in hyprland.conf
5. **Apps** — yay (AUR), firefox, alacritty, discord, spotify, claude-code (via npm)
6. **Dev/AI** — python, docker, ripgrep/fd/fzf/bat/eza/zoxide/lazygit/tmux, PyTorch+CUDA, ollama-cuda
7. **Shell** — zsh + starship, JetBrains Mono Nerd font
8. **Optimizations** — fstrim timer, tlp, pacman parallel downloads, ufw firewall, timeshift btrfs snapshots

### Notes from install
- `mkfs.fat -F32 -I` was needed (existing partition table required `-I` override)
- Disk is /dev/nvme0n1 (p1=EFI, p2=swap, p3=root btrfs)
