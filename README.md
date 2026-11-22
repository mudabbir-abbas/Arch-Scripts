<!-- ims-managed -->
# Arch-Scripts

My personal utility scripts from `~/.local/bin` on Arch Linux.

This repo is **not a product and not a tool distribution.** It's a window into how I set up and look after my own machine. If something here is useful to you, take it. If you have a better idea, tell me.

---

## ⚠️ Please read first

### These scripts are AI-written ("vibecoded")

Every script in this repo was written with AI assistance (Claude), through conversation and iteration, not hand-coded line by line. My role is describing what I want, testing on my own machine, reporting what's broken, and deciding what stays.

What that means for you:

- **Treat the code as unreviewed by a professional.** It works for me, but "works on my laptop" is the only guarantee there is.
- **Read a script before you run it**, especially the ones that change things on your system (see the table below).
- If you spot a bug, a risky pattern or something silly, I genuinely want to hear about it.

### They're built for one machine

Everything here is tuned to my setup:

| Part | What I use |
|---|---|
| Laptop | Dell Latitude 5400 (i5-8350U, 8 GB RAM, 256 GB NVMe) |
| OS | Arch Linux |
| Desktop | Niri + Dux Shell |
| Boot | systemd-boot, Plymouth |
| Disk | Btrfs + Snapper, zram |
| Power | power-profiles-daemon |

Some scripts assume parts of this (for example `syscheck` and `stage` are very specific). Others are generic and should work on most Linux systems. The table below says which is which.

### No warranty, no support

Provided as-is. I'm not promising support, fixes, or that anything will keep working. Use at your own risk, and back up before trying anything that touches your files or config.

---

## The scripts

| Script | What it does | Portability | Changes your system? |
|---|---|---|---|
| [`appimgr`](#appimgr) | Integrates AppImages into the app launcher; add, update from GitHub and remove one or several at once, roll back, optionally clean up leftover data | Most Linux desktops | Yes (`~/.local/share`) |
| [`fbackup`](#fbackup) | Pack chosen files/folders into one archive and restore them to their original locations | Any Linux with GNU tar | Yes (writes/restores files) |
| [`ims`](#ims) | "Install My Scripts": moves updated scripts from `~/Downloads` (or any path) to `~/.local/bin` | Anywhere | Yes (replaces scripts) |
| [`stage`](#stage) | Save Niri window layouts as one-key profiles and restore them | **Niri only** | Yes (edits Niri config) |
| [`syscheck`](#syscheck) | Read-only health report for Arch + Niri + DMS | **Arch, tuned to my laptop** | No (read-only) |
| [`thermal-toast`](#thermal-toast) | Desktop notification when the CPU stays hot too long; can install itself as a user service | Intel `coretemp` machines | Only if you run `install` (user service + settings file) |

---
 
### appimgr

Makes AppImages behave like installed apps.

- Adds AppImages (local files, direct links, GitHub release links, or `--repo OWNER/REPO`), extracts the icon and `.desktop` entry, and creates a launcher entry. Several files or links can be given at once (`appimgr add a.AppImage b.AppImage`).
- **Several apps at once:** `appimgr update app1 app2 app3` updates only those, `appimgr remove app1 app2 app3` removes them with one question. Without names, `update` covers every app that has an update source.
- Checks for updates from GitHub releases or a direct URL, in parallel, lists them in a table (old → new version, size, total), then asks once before installing.
- **Package-manager style output:** a progress bar while checking, one live progress line per download (name, bar, percentage, size, speed), and one line per app while installing.
- Verifies SHA-256 digests when GitHub provides them.
- Keeps the previous version for `rollback`.
- `remove` moves the AppImage to the Trash instead of deleting it.
- **Leftover data:** `remove --purge-data` searches `~/.config`, `~/.cache`, `~/.local/share`, `~/.local/state` and hidden folders in `~` for entries named like the app, shows them with sizes, and lets me pick what to delete. It's a name match, so it's a list to review, never an automatic deletion.
- **Portable mode** (`add --portable`, or `portable <slug> on|off|status`) keeps all of an app's data in `<slug>.AppImage.home` next to the AppImage, so removing a temporary app leaves nothing behind and `--purge-data` deletes it exactly, with no guessing.
- Also: custom icons (`icon`), manual update sources (`source`), launching from a terminal (`run`), and an interactive menu when run with no arguments (it accepts several numbers, like `1 3 4`).
- Downloads with `curl`, several at a time (`-j N`).
- Honors `GITHUB_TOKEN` (raises the API limit); the token is only sent to `api.github.com` and removed from the environment for child processes.

**Needs:** `bash`, `curl`. **Optional:** `fuse2` (without it, extract-and-run mode is used).
**Writes to:** `~/.local/share/AppImages` (including `.AppImage.home` folders in portable mode), `~/.local/share/applications`.

### fbackup

Picks files/folders, packs them into one archive, and puts them back in their original locations later.

- Paths inside your home are stored as `HOME/...`, so a restore works even if your username changes.
- Compression: `zstd` if installed, otherwise `gzip`, otherwise plain tar.
- Existing files are **not** overwritten unless you pass `--overwrite`.
- Saved path list in `~/.config/fbackup/paths.conf` for things you always back up (`fbackup config add|remove|list|edit`).
- `--home` mode never needs root; the default mode allows any path and needs root. On restore it checks the archive first: home-only archives need no root, archives with items outside the home do (and it offers to re-run itself with `sudo`).
- Restore preserves existing symlinked folders and writes through them. It warns when archive paths will follow those links; only restore archives you trust, especially when restoring system files as root.
- `fbackup list ARCHIVE` shows what's inside an archive; `fbackup --check` reports which tools are installed.
- Verifies the archive after creating it, and reports unreadable files and symlinks.
- Run with no arguments, it opens a small menu.

**Needs:** `bash` 4+, GNU `tar`. **Optional:** `zstd`, `gzip`.

> ⚠️ This one writes files. Try `fbackup list ARCHIVE` and a restore into a test setup before trusting it with anything important. Snapshots and backups are not the same thing, and this is a personal tool, not a backup product.

### ims

"Install My Scripts". When I download a new version of a script from the AI chat, `ims` drops it into place.

- Matches `~/Downloads` files to installed scripts by name (handles `foo.sh`, `foo (1).sh`, etc.).
- **Install from anywhere:** an argument containing a slash is a path to the file itself, e.g. `ims ~/code/foo.py` installs it as `foo`. It's moved like Downloads files (`-c`/`--copy` keeps the original).
- Backs up the old copy (keeps the last 3), runs `chmod +x`, and replaces the script atomically. It can update itself safely.
- Plain files like `PLANS.md` are handled too (never made executable); plain `ims` only replaces them when the Downloads copy carries an `ims-managed` marker.
- `ims list` shows installed scripts, versions, and updates waiting in Downloads.
- `ims restore <name>` rolls back to the latest backup (run again to toggle).
- `ims completions --install` sets up fish tab completion (Downloads files, subcommands, ordinary paths).
- `--dry-run` shows what would happen without changing anything.

**Needs:** `bash` and coreutils. **Optional:** `fish` for the completions.

### stage

Saves your Niri window layout as a named profile ("stage") and restores it later with one shortcut.

- Captures open windows, their workspace, column/tile position, size and floating state. `stage capture NAME` saves everything without prompts (except your own terminal window).
- Restores by launching missing apps, matching windows, and moving them into place. Windows saved without an app ID or launch command are skipped and listed as problems.
- Can open specific URLs in browser windows and add extra launch arguments.
- Generates Niri keybind includes, validates with `niri validate` before and after every change, and **rolls back** automatically on failure.
- Keeps snapshots of your config before changes, and warns about shortcut conflicts.
- Interactive interface built on `gum`; `--dry-run` shows what restore would do.

**Needs:** `niri`, `python3`, `gum` (for the interface). **Niri only.**
**Edits:** one `include` line in `~/.config/niri/config.kdl` and the folder `~/.config/niri/stage/`.

### syscheck

A reusable, **read-only** health report for my exact setup: Arch + Niri + Dank Material Shell, systemd-boot/UKI, Btrfs + Snapper.

- Checks boot time, failed units, journal errors, coredumps, CPU/thermals/throttling, memory and swap, NVMe SMART and Btrfs health, battery, network, Bluetooth, pending updates, package integrity, firewall, firmware updates, and more.
- Three output modes: default (healthy checks are one line), `--summary`, and `--full`. `-d DIR` chooses where the report is saved.
- Every check has a time limit (overridable with `DL_LOCAL`, `DL_SLOW`, `DL_NET`, `NET_BUDGET`), so one hung check can't stall the report.
- **Masks personal data before saving:** hostname, username, LAN IPs, MAC addresses, Wi-Fi/VPN names, serials, machine-id, UUIDs. It's still worth skimming a report before sharing it.

**Needs:** Arch Linux, `sudo`; helpers like `smartmontools`, `lm_sensors`, `upower` are reported as missing if absent.
**Very machine-specific.** It contains checks for my hardware, e.g. a known quirk of my NVMe model and known upstream bugs affecting my setup. You'll want to edit it for yours.

### thermal-toast

A tiny watcher that sends a desktop notification when the CPU package stays above a temperature for too long. It can also manage itself as a systemd user service, so there's no unit file to write by hand.

- Defaults: alert above 90 °C for 60 s, 10-minute cooldown, checks every 5 s.
- Settings come from, in priority order: option, `THERMAL_TOAST_*` variable, settings file (`~/.config/thermal-toast/config`), default.
- The notification lists the top CPU-using processes.
- **Service commands:** `install`, `uninstall [--purge]`, `start`, `stop`, `restart`, `enable`, `disable`, `reload`, `status`, `log [-f]`, `config set NAME VALUE` (reloads a running service), `test`, and `run` to watch in the terminal.
- With no arguments on a terminal it opens a menu. Without a terminal, or with only options, it watches (so an old autostart line keeps working).

**Needs:** `notify-send` (`libnotify`), an Intel `coretemp` (or `x86_pkg_temp`) sensor, and a systemd user session for the service commands.
**Writes to (only after `install` / `config set`):** `~/.config/systemd/user/thermal-toast.service`, `~/.config/thermal-toast/`.

---

## 💬 This is open. Talk to me.

These are **my personal scripts**, and they will keep changing as I change my setup. But this repo is public on purpose:

- **Take anything.** Copy a script, a function, an idea. Rip out the part you like. No need to ask.
- **Change anything.** Fork it, adapt it to your hardware or desktop, make it your own.
- **Recommend things.** Better approach? Safer pattern? A tool I should use instead? Open an issue or a discussion. I'd rather learn something than defend the code.
- **Point out problems.** Bugs, risky behavior, or things that only work on my laptop: say so.
- **Pull requests are welcome**, but I may not merge everything. My scripts have to fit my machine, and I might take your idea and implement it my own way. Please don't be offended by that.

I can't promise quick answers or any support. This is a hobby setup, not a project with a roadmap.

If you build something off these scripts, I'd love to see it, though you don't have to tell me.

---

## License

Released under the [MIT License](LICENSE). Do what you want with it; just keep the notice, and remember there's no warranty.

## Disclaimer (again, because it matters)

AI-generated code, tested only on my own machine, provided as-is. Read before you run. Back up before you restore. Don't point a script that changes your system at something you can't afford to lose without understanding what it does first.
