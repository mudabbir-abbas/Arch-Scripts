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
- **Don't assume edge cases are handled.** Many were found and fixed by running into them, which is what the "NEW IN THIS VERSION" headers record.
- If you spot a bug, a risky pattern or something silly, I genuinely want to hear about it.

### They're built for one machine

Everything here is tuned to my setup:

| Part | What I use |
|---|---|
| Laptop | Dell Latitude 5400 (i5-8350U, 8 GB RAM, 256 GB NVMe) |
| OS | Arch Linux |
| Desktop | Niri + Dank Material Shell, `greetd` + `dms-greeter` |
| Boot | systemd-boot, UKI, Plymouth |
| Disk | Btrfs + Snapper, zram, no disk encryption, no Secure Boot |
| Power | power-profiles-daemon |

Some scripts assume parts of this (for example `syscheck` and `stage` are very specific). Others are generic and should work on most Linux systems. The table below says which is which.

### No warranty, no support

Provided as-is. I'm not promising support, fixes, or that anything will keep working. Use at your own risk, and back up before trying anything that touches your files or config.

---

## The scripts

| Script | What it does | Portability | Changes your system? |
|---|---|---|---|
| [`appimgr`](#appimgr) | Integrates AppImages into the app launcher; update from GitHub, roll back, remove (optionally with leftover data) | Most Linux desktops | Yes (`~/.local/share`) |
| [`binvc`](#binvc) | Friendly git helper for the scripts repo itself (the older, single-repo version of `repos`) | Anywhere with git | Yes (git commits/pushes) |
| [`fbackup`](#fbackup) | Pack chosen files/folders into one archive and restore them to their original locations | Any Linux with GNU tar | Yes (writes/restores files) |
| [`ims`](#ims) | "Install My Scripts": moves updated scripts from `~/Downloads` (or any path) to `~/.local/bin` | Anywhere | Yes (replaces scripts) |
| [`repos`](#repos) | Git helper for several repos: set up, switch, link to GitHub, remove, plus everyday git | Anywhere with git (`gh` optional) | Yes (git commits/pushes, **can delete repo folders**) |
| [`shelf`](#shelf) | Menu launcher for every script in `~/.local/bin` | Anywhere (nicer with `fzf`) | No |
| [`stage`](#stage) | Save Niri window layouts as one-key profiles and restore them | **Niri only** | Yes (edits Niri config) |
| [`syscheck`](#syscheck) | Read-only health report for Arch + Niri + DMS | **Arch, tuned to my laptop** | No (read-only) |
| [`thermal-toast`](#thermal-toast) | Desktop notification when the CPU stays hot too long; can install itself as a user service | Intel `coretemp` machines | Only if you run `install` (user service + settings file) |

All scripts follow the same conventions:

- `VERSION="MAJOR.MINOR.PATCH"` near the top, shown by `--version` and at the top of `--help`.
- A **NEW IN THIS VERSION** comment block at the top: a short changelog for the latest bump.
- Run with no arguments, they still do something useful (a menu, or the help text).
- A `# DESC:` line near the top that `shelf` uses as the description.

---

### appimgr

Makes AppImages behave like installed apps.

- Adds an AppImage (local file, direct link, GitHub release link, or `--repo OWNER/REPO`), extracts the icon and `.desktop` entry, and creates a launcher entry.
- Checks for updates from GitHub releases or a direct URL, in parallel, then asks once before installing.
- Verifies SHA-256 digests when GitHub provides them.
- Keeps the previous version for `rollback`.
- `remove` moves the AppImage to the Trash instead of deleting it.
- **Leftover data:** `remove --purge-data` searches `~/.config`, `~/.cache`, `~/.local/share`, `~/.local/state` and hidden folders in `~` for entries named like the app, shows them with sizes, and lets me pick what to delete. It's a name match, so it's a list to review, never an automatic deletion.
- **Portable mode** (`add --portable`, or `portable <slug> on|off|status`) keeps all of an app's data in `<slug>.AppImage.home` next to the AppImage, so removing a temporary app leaves nothing behind and `--purge-data` deletes it exactly, with no guessing.
- Also: custom icons (`icon`), manual update sources (`source`), launching from a terminal (`run`), and an interactive menu when run with no arguments.
- Uses `aria2c` for downloads if installed, otherwise `curl`.
- Honors `GITHUB_TOKEN` (raises the API limit); the token is only sent to `api.github.com` and removed from the environment for child processes.

**Needs:** `bash`, `curl`. **Optional:** `aria2c`, `fuse2` (without it, extract-and-run mode is used).
**Writes to:** `~/.local/share/AppImages` (including `.AppImage.home` folders in portable mode), `~/.local/share/applications`.

### binvc

A small, friendly wrapper around git for the `~/.local/bin` scripts repo.

> `repos` (below) is the newer version: it does everything `binvc` does and can also manage several repos. I'm keeping `binvc` around for now in case I don't get on with `repos`. It only knows one repo.

- `status` suggests the next best command (commit, push, pull, sync) based on local and remote state.
- `sync "message"` commits, pulls with rebase and pushes in one go.
- Warns about possible secrets before committing and can scan the whole tracked tree.
- Warns about scripts that have a shebang but aren't executable.
- Has restore / old-version / undo helpers with confirmation prompts.
- Interactive menu when run with no arguments (an error in one action returns to the menu instead of closing it).

**Needs:** `git`. The repo path defaults to `~/.local/bin` (override with `BINVC_REPO`).

### fbackup

Picks files/folders, packs them into one archive, and puts them back in their original locations later.

- Paths inside your home are stored as `HOME/...`, so a restore works even if your username changes.
- Compression: `zstd` if installed, otherwise `gzip`, otherwise plain tar.
- Existing files are **not** overwritten unless you pass `--overwrite`.
- Saved path list in `~/.config/fbackup/paths.conf` for things you always back up (`fbackup config add|remove|list|edit`).
- `--home` mode never needs root; the default mode allows any path and needs root. On restore it checks the archive first: home-only archives need no root, archives with items outside the home do (and it offers to re-run itself with `sudo`).
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

### repos

The successor to `binvc`: a git helper that keeps a list of my repos, with one "active" repo that the everyday commands act on.

- **Everyday git, same as `binvc`:** `status` (suggests the next command), `commit`, `push`, `pull`, `sync "message"`, `log`, `undo`, `tag`, secret scan, and so on. These act on the active repo, and commands that change something print which repo they are working on.
- **Set up a repo:** `repos new NAME` runs `git init`, creates a starter `.gitignore`, makes the first commit (with the secret check) and registers the repo. It can then create the GitHub repo with `gh` (private by default), or link an existing URL, or stay local. `repos add` registers a folder I already have, `repos clone URL` clones one.
- **Switch repos:** `repos use NAME` (no name = pick from a list). `repos list` shows every repo with its branch and whether it has changes to commit, push or pull.
- **GitHub link:** `repos link` / `repos link --github` / `repos unlink` for the active repo.
- **Remove a repo:** `repos forget NAME` only drops it from the list and keeps all files. `repos remove NAME` unlinks it from GitHub and then **deletes the whole folder** (including `.git` and untracked files, no Trash). It shows what will be lost, needs the repo name typed to confirm, and refuses to run without a terminal. The GitHub repo itself is never deleted.
- On first run my existing `~/.local/bin` repo is registered as `scripts`. The list lives in `~/.config/repos/`.
- Interactive menu when run with no arguments.

**Needs:** `git`. **Optional:** `gh` (GitHub CLI, `sudo pacman -S github-cli` then `gh auth login`) to create GitHub repos.
`BINVC_REPO` and `REPOS_REPO` override the active repo for a single run.

> ⚠️ `repos remove` deletes things for good. Read the summary it prints before typing the name, and use `forget` if you only want the repo out of the list.

### shelf

A launcher menu for everything in `~/.local/bin`.

- Lists scripts with their version and a description (read from the first lines of each script, never executed).
- With `fzf`: Enter to run, `Ctrl-A` to run with arguments, `Ctrl-H` for `--help`, `Ctrl-E` to edit, `Tab` for an info panel.
- Without `fzf`: falls back to a plain numbered menu.
- Ctrl+C inside a launched script brings you back to the menu instead of closing it.
- `-l`/`--list` prints the table, `--info NAME` prints one script's info card, `-d DIR` (or `SHELF_DIR`) scans another folder. The table is also printed when there's no terminal (e.g. `shelf | cat`).

**Needs:** `bash`. **Optional:** `fzf`.

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

- Defaults: alert above 85 °C for 60 s, 10-minute cooldown, checks every 5 s.
- Settings come from, in priority order: option, `THERMAL_TOAST_*` variable, settings file (`~/.config/thermal-toast/config`), default.
- The notification lists the top CPU-using processes.
- **Service commands:** `install`, `uninstall [--purge]`, `start`, `stop`, `restart`, `enable`, `disable`, `reload`, `status`, `log [-f]`, `config set NAME VALUE` (reloads a running service), `test`, and `run` to watch in the terminal.
- With no arguments on a terminal it opens a menu. Without a terminal, or with only options, it watches (so an old autostart line keeps working).

**Needs:** `notify-send` (`libnotify`), an Intel `coretemp` (or `x86_pkg_temp`) sensor, and a systemd user session for the service commands.
**Writes to (only after `install` / `config set`):** `~/.config/systemd/user/thermal-toast.service`, `~/.config/thermal-toast/`.

---

## How the pieces fit together

```
 AI chat ──► ~/Downloads ──► ims ──► ~/.local/bin ──► shelf (menu)
                                          │
                                          └──► repos (or binvc) ──► this repo
```

My workflow: ask the AI for a change, download the updated script, run `ims` to install it (or `ims path/to/file` if it landed somewhere else), try it, and when I'm happy run `repos sync` (or the older `binvc sync`) to commit and push.

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
