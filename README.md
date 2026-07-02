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
  main repository. Nothing in this project pins versions: every run
  installs/upgrades to the newest available release of everything
  (including umu-launcher, resolved from its latest GitHub release).
- **`flathub` role**: flatpak + the flathub remote, shared by the two
  package roles below.
- **`packages_base` role**: everything from install-guide.md's
  post-install parts — DNS fix, NVIDIA driver, multilib, zsh with a
  minimal managed config (no framework; personal tweaks go in
  `~/.zshrc.local`), Hyprland + example config, TTY1 autologin,
  Neovim/Fastfetch, Zen Browser, NTP, boot-time tweaks.
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
- **The xbps hang, properly diagnosed** (2026-07-02): `xbps-install -S`
  can wedge forever mid-sync with `CLOSE-WAIT` sockets — the server
  closed the connection, unread bytes sit in the receive queue, and
  xbps's bundled libfetch blocks in a read with no timeout. Earlier
  theories are wrong: it is *not* IPv6 (a plain Python client fetched
  the same repodata over IPv6 from the same box in 0.2s while xbps hung)
  and *not* Fastly (it was observed live hanging against
  `repo-de.voidlinux.org`, a direct mirror, and `repo-default` is just a
  CNAME to the `repo-fi` origin — not a CDN edge). xbps 0.60.7 has no
  timeout setting, so the playbook defends itself instead: every
  index-refresh/upgrade goes through a timeout-kill-retry wrapper
  (`roles/system/tasks/xbps_sync.yml`,
  `roles/packages_base/tasks/refresh_index.yml`). The `system` role also
  points the nonfree/multilib sub-repositories at the chosen fastest
  mirror — by default those silently kept using `repo-default` because
  only the main repo conf was overridden.
