# void.config

Ansible playbook that finishes configuring a box installed by
[void.install](https://github.com/stivce/void.install): package updates,
timezone/keymap, fastest mirror, then everything from `install-guide.md`
and `gaming.md` beyond what the base installer already does. Runs fully
unattended — no prompts, every feature below is always applied.

## What it does

- **`system` role**: syncs the package index and upgrades xbps itself,
  upgrades all installed packages, ensures the timezone and keymap,
  benchmarks a list of candidate mirrors and sets the fastest one as the
  main repository.
- **`packages_base` role**: everything from install-guide.md's
  post-install parts — DNS fix, NVIDIA driver, multilib, ZSH + Oh My Zsh,
  Hyprland + example config, TTY1 autologin, Neovim/Fastfetch, Zen
  Browser, NTP, boot-time tweaks.
- **`packages_gaming` role**: everything from gaming.md — GameMode/
  MangoHud, Steam, ProtonPlus, Prism Launcher, World of Warcraft via
  umu-launcher + a generated Battle.net launcher script, Discord, and
  CurseForge (prepares `~/.local/bin`/PATH; the AppImage itself still has
  to be downloaded manually — there's no stable direct-download URL for
  it).

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

   You'll be prompted for your sudo password once up front; everything
   else runs unattended.

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
- The `system` role's very first task adds a precedence rule to
  `/etc/gai.conf` so IPv4 is preferred over IPv6 for name resolution.
  This works around a real, reproducible hang: `xbps-install -S` would
  connect to a mirror over IPv6, then stall forever mid-transfer (visible
  as a `CLOSE-WAIT` socket that never closes) — consistent with an IPv6
  MTU black hole somewhere on the path. IPv4 to the same mirrors worked
  fine. If `xbps-install` ever hangs again after this, check `ps`/`ss` on
  the target for the same `CLOSE-WAIT` pattern before assuming it's just
  slow.
