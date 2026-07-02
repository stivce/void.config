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
  minimal managed config for root (no framework; the user's zsh config
  comes from the dotfiles role), Hyprland, TTY1 autologin,
  Neovim/Fastfetch, Zen Browser, NTP, boot-time tweaks.
- **`packages_desktop` role**: everything the
  [void.dot](https://github.com/stivce/void.dot) dotfiles assume at
  runtime — hyprlock/hypridle/hyprpolkitagent/hyprshot (from the
  hyprland-void repo), kitty, SwayNotificationCenter, awww, quickshell,
  matugen, pavucontrol, nautilus, qt5ct, OBS, the script dependencies
  (grim, slurp, wl-clipboard, playerctl, brightnessctl, jq, libnotify,
  ImageMagick, starship, fzf), the JetBrainsMono Nerd Font (latest
  GitHub release, installed once), the Volantes Light cursor theme
  (`XCURSOR_THEME` in the dotfiles; not packaged anywhere, so it's built
  from upstream source once and symlinked under the name the dotfiles
  expect), and NetworkManager — which replaces
  the installer's dhcpcd, because the dotfiles' network menu and swaync
  drive everything through `nmcli`. The dhcpcd→NM handoff runs
  fire-and-forget so the playbook survives the momentary interface
  reconfiguration of its own SSH connection.
- **`packages_gaming` role**: everything from gaming.md — GameMode/
  MangoHud, Steam, ProtonPlus, Prism Launcher, World of Warcraft via
  umu-launcher + a generated Battle.net launcher script, Discord, and
  CurseForge (prepares `~/.local/bin`/PATH; the AppImage itself still has
  to be downloaded manually — there's no stable direct-download URL for
  it).
- **`dotfiles` role** (runs last): clones
  [void.dot](https://github.com/stivce/void.dot) into `~/.dotfiles`
  (once — never force-updated, since the symlinks make that checkout the
  live config) and symlinks its contents into place: every entry of
  `.config/`, `.local/bin/`, `.local/share/` and `Pictures/`, plus
  `~/.zshrc` and `~/.gitconfig`. Real files already at a destination
  (e.g. the generated fastfetch config, or the previously managed
  `~/.zshrc`) are moved aside to `<name>.pre-dotfiles`, never deleted.
  `~/.dotfiles` is the fixed path because hyprland.conf and waybar
  reference it directly.

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
- **The xbps "hang", properly diagnosed** (2026-07-02): it was never a
  network problem. `xbps-install -S` prompts on **stdin** before
  trusting a new repository signing key — and deliberately ignores `-y`
  for that prompt. Under ansible, stdin is a pipe that never answers,
  so the process blocked forever the moment the hyprland-void repo
  (signed by `hyprland-void-github-action`, an unknown key) entered the
  config. Everything else was a red herring chased across three
  workaround commits: not IPv6, not Fastly, not libfetch — the
  `CLOSE-WAIT` sockets with unread bytes were just idle keep-alive
  connections whose servers gave up while xbps sat waiting at the
  prompt (plain-client fetches of the same URLs succeeded instantly the
  whole time). The fix: `packages_base` imports the repo key explicitly
  (`yes y | xbps-install -S`, guarded by a fingerprint check) before
  any module-driven sync can meet the prompt. The timeout-kill-retry
  wrappers (`roles/system/tasks/xbps_sync.yml`,
  `roles/packages_base/tasks/refresh_index.yml`) stay as defense in
  depth — a future unknown key or a genuinely dead mirror now costs
  minutes, not an infinite hang. The `system` role also points the
  nonfree/multilib sub-repositories at the chosen fastest mirror; those
  silently kept using `repo-default` before, because only the main repo
  conf was overridden.
