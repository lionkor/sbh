# sandbox-here

Run a command isolated to the current directory — no access to the rest of
your home, no network, no other writable paths except the app's own config
files.

## What it does

`sandbox-here` wraps [bubblewrap](https://github.com/containers/bubblewrap) to
create a Linux-namespace container where:

| Path | Access | Notes |
|------|--------|-------|
| `$PWD` (as `/work`) | **read-write** | The only data you can modify |
| App's own configs | **read-write** | Auto-discovered (see below) |
| Other dotfiles in `$HOME` | read-only | Other apps' settings, visible but safe |
| Non-dot directories in `$HOME` | **hidden** | `~/src`, `~/Downloads`, `~/Documents`, etc. don't exist |
| `/usr`, `/bin`, `/etc`, … | read-only | Needed to run binaries |
| `/tmp`, `/run` | private tmpfs | Empty, scrubbed on exit |
| Network | **none** | `--unshare-all` includes net namespace |
| `/root`, `/mnt`, `/media`, `/var`, `/opt` | **hidden** | Not mounted |

## Usage

```bash
# Open a shell isolated to the current directory
sandbox-here

# Run any command sandboxed
sandbox-here vim file.txt
sandbox-here make
sandbox-here npm install
sandbox-here python -m http.server
```

## Dependencies

- **bubblewrap** (`bwrap`) — available in all major distros (`apt install bubblewrap`, `dnf install bubblewrap`, `pacman -S bubblewrap`)
- **Python 3.6+** — standard library only (`os`, `shutil`, `sys`, `pathlib`)

## How config discovery works

When you run `sandbox-here <command>`, the script:

1. **Extracts the app name** from the command (basename of the first argument).

2. **Resolves symlinks** through `$PATH` — if `sh` is a symlink to `bash`,
   both `"sh"` and `"bash"` are used as app names.

3. **Checks well-known config paths** for each name:

   | Pattern | Example for `nvim` |
   |---------|-------------------|
   | `~/.config/<app>/` | `~/.config/nvim/` |
   | `~/.cache/<app>/` | `~/.cache/nvim/` |
   | `~/.local/share/<app>/` | `~/.local/share/nvim/` |
   | `~/.local/state/<app>/` | `~/.local/state/nvim/` |
   | `~/.<app>/` | `~/.nvim/` |
   | `~/.<app>rc` | `~/.nvimrc` |

   Only paths that actually exist on the host are mounted.

4. **Everything else** under `$HOME/.*` is mounted read-only; non-dot
   directories are never mounted and therefore invisible inside the sandbox.

### Why symlink resolution matters

```bash
$ ls -l /usr/bin/sh
lrwxrwxrwx 1 root root 4 … /usr/bin/sh -> bash
```

`sandbox-here sh` will grant write access to `~/.bashrc` and
`~/.config/bash/` because the real binary behind `sh` is `bash`.

## Mount layering — why read-only comes before writable

bwrap mounts are processed in order; later mounts override earlier ones at
the same path or subtree.  The script mounts things in this order:

```
1. --tmpfs $HOME           ← empty home
2. --ro-bind-try  each other dotfile/dotdir
3. --bind-try      each app-owned config path   ← punches through step 2
```

Without this ordering, a read-only `~/.config` parent mount would shadow
the writable `~/.config/nvim/` child mount, making it impossible for the
app to write its own configs.  By mounting read-only parents first and
writable children second, the writable bind "punches through" and wins.

## Filesystem layout inside the sandbox

```
/                        ← only system dirs + tmpfs mounts
├── usr/                 (ro)
├── bin/                 (ro)
├── etc/                 (ro)
├── dev/                 (private)
├── proc/                (private)
├── tmp/                 (private, empty)
├── run/                 (private, empty)
├── home/<user>/         (tmpfs + selective bind mounts)
│   ├── .config/nvim/    (rw)  ← app's own config
│   ├── .config/git/     (ro)  ← other app, visible but safe
│   ├── .bashrc          (ro or rw, depending on which app is running)
│   ├── .ssh/            (ro)
│   └── Downloads/       DOES NOT EXIST
└── work/                (rw)  ← your $PWD
```

## Design decisions

**Why Python instead of shell?**  The config-discovery logic (symlink
resolution, path classification, mount ordering) outgrew zsh.  Python's
`pathlib` and `shutil.which` make the code readable and correct.

**Why bubblewrap instead of firejail/systemd-nspawn?**  bubblewrap is
unprivileged (uses user namespaces), has no SUID binary, gives precise
control over every mount point, and is available everywhere.

**Why auto-discover dotfiles instead of a hardcoded list?**  A hardcoded
allowlist would rot as you install new tools.  Globbing `$HOME/.*` at
runtime automatically picks up every config file, present and future.

**Why separate `/work` instead of mounting `$PWD` in-place?**  If `$PWD` is
under `$HOME` and we mount `$HOME` as tmpfs, we'd need to punch a writable
hole at the exact right path.  Using a separate `/work` mount point avoids
this conflict entirely.

**Why `--die-with-parent`?**  If the outer shell is killed, the sandbox
must be torn down too.  Without this flag, bwrap namespaces can outlive
the parent process and leak resources.

## Files

```
~/.config/sandbox-here/
├── sandbox-here    ← the Python script (executable)
└── README.md       ← this file
```

A small zsh wrapper function in `~/.zshrc` calls the script:

```zsh
sandbox-here() {
    "$HOME/.config/sandbox-here/sandbox-here" "$@"
}
```
