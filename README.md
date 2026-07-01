# void.config

Ansible playbook that finishes configuring a box installed by
[void.install](https://github.com/stivce/void.install): package updates,
timezone/keymap, fastest mirror, then everything from `install-guide.md`
and `gaming.md` beyond what the base installer already does.

## What it does

- **`system` role** (non-interactive): syncs the package index and
  upgrades xbps itself, upgrades all installed packages, ensures the
  timezone and keymap, benchmarks a list of candidate mirrors and sets
  the fastest one as the main repository.
- **`packages_base` role** (interactive): asks `y`/`N` for each
  install-guide.md feature — DNS fix, NVIDIA driver, multilib, ZSH (+
  optional Oh My Zsh), Hyprland (+ example config), TTY1 autologin,
  Neovim/Fastfetch, Zen Browser, NTP, boot-time tweaks — and only applies
  what you say yes to.
- **`packages_gaming` role** (interactive): same pattern for
  gaming.md — GameMode/MangoHud, Steam, ProtonPlus, Prism Launcher,
  World of Warcraft via umu-launcher + a generated Battle.net launcher
  script, Discord, and CurseForge (prepares `~/.local/bin`/PATH; the
  AppImage itself still has to be downloaded manually — there's no
  stable direct-download URL for it).

Parts 1-14 of `install-guide.md` (partitioning through first boot) and
Part 16 (mirror selection) are deliberately not repeated here — the first
is void.install's job, the second is handled by the `system` role
instead. Part 7 of `gaming.md` (The Archon via Lutris) is a manual GUI
walkthrough and isn't automated.

## Requirements

- A box already installed via void.install, reachable over SSH, with
  Python 3 on it (void.install installs this by default)
- `ansible-core` plus the `community.general` and `ansible.posix`
  collections on your control machine:

  ```sh
  ansible-galaxy collection install community.general ansible.posix
  ```

## Usage

1. Copy the example inventory and fill in your host:

   ```sh
   cp inventory/hosts.yml.example inventory/hosts.yml
   ```

   ```yaml
   all:
     hosts:
       voidbox:
         ansible_host: 192.168.1.50
         ansible_user: alice
   ```

2. Adjust `group_vars/all.yml` if you want a different timezone/keymap or
   mirror candidate list than the defaults (Vienna/`us`/Tier 1 + nearby
   European mirrors).

3. Run it:

   ```sh
   ansible-playbook site.yml --ask-become-pass
   ```

   You'll be prompted for your sudo password once up front, then for
   each optional feature as the `packages_base` and `packages_gaming`
   roles reach their prompts. The `system` role runs first with no
   prompts at all.

## Notes

- `ansible-lint` is clean except for one unavoidable false positive:
  it can't resolve `ansible.posix.mount` in this repo's local
  environment (a collection-path quirk in `ansible-lint` itself), even
  though `ansible-playbook --syntax-check` confirms the module resolves
  fine at actual runtime.
- The `boot_optimization` feature only updates `/etc/fstab` with the
  btrfs performance mount options — it deliberately doesn't force a live
  remount of `/`, so a reboot is needed to apply them.
- `inventory/hosts.yml` is gitignored, same pattern as void.install's
  `void.cfg` — only the `.example` file is tracked.
