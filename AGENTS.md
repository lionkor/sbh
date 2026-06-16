# sandbox-here — agent reference

`sbh` wraps bubblewrap to sandbox a command.  The agent (pi, codex, etc.)
runs inside it; the agent's own config is writable, everything else is
locked down.

## Sandbox layout

```
/                        (tmpfs root + bind mounts)
├── usr, bin, sbin, etc  (ro-bind from host)
├── lib, lib64            (ro-bind-try from host)
├── etc/hosts            (ro-bind-data overlay — only localhost entries)
├── dev/                  (--dev: private /dev)
├── proc/                 (--proc: private /proc)
├── tmp/, run/            (tmpfs, empty, scrubbed on teardown)
├── home/$USER/           (tmpfs + selective binds)
│   ├── .config/$APP/    (rw, if APP matches the command)
│   ├── .cache/$APP/     (rw, same condition)
│   ├── .local/share/$APP/  (rw)
│   ├── .local/state/$APP/  (rw)
│   ├── .$APP/            (rw)
│   ├── .$APPrc           (rw)
│   ├── everything else   (ro-bind-try — unless blocked)
│   └── sensitive paths   (not mounted — see blocklist below)
└── w/<full-path-to-$PWD>/  (rw-bind from $PWD)
```

## App config discovery

The script extracts the app name from the command (basename), resolves
symlinks through `$PATH`, and matches against these patterns:

| Pattern | Example (`nvim`) |
|---------|-----------------|
| `~/.config/<app>/` | `~/.config/nvim/` |
| `~/.cache/<app>/` | `~/.cache/nvim/` |
| `~/.local/share/<app>/` | `~/.local/share/nvim/` |
| `~/.local/state/<app>/` | `~/.local/state/nvim/` |
| `~/.<app>/` | `~/.nvim/` |
| `~/.<app>rc` | `~/.nvimrc` |

Only paths that actually exist on the host are mounted.

Symlink resolution: if `/usr/bin/sh` → `bash`, both `sh` and `bash` are
checked, so `~/.bashrc` and `~/.config/bash/` become writable.

## Sensitive path blocklist

These paths are never mounted — not even read-only:

```
.ssh .gnupg .codex .pki .docker .mozilla .thunderbird .librewolf
.aws .azure .gcloud .gsutil .copilot .electrum .tor .gnome .kde4 .mcli
```

Exception: if the command's app name matches the blocked directory (e.g.
`sandbox-here codex` → `~/.codex/`), the app's own config wins and is
mounted read-write.  The blocklist only gates what *other* apps can see.

Defined as `_SENSITIVE_DOTFILES` in the script.

## Mount ordering

```
1. --tmpfs $HOME                (empty home)
2. --ro-bind-try <safe dotfiles>  (skip blocked paths)
3. --bind-try <app configs>       (writable, overrides step 2)
4. --bind-try <container sockets> (--docker / --podman / --containerd)
5. --ro-bind or --bind <extra>    (--add ro|rw SRC DST)
6. --bind $PWD /w/<full-path>       (workspace)
```

Later mounts override earlier ones at the same path, so step 3 punches
through any read-only parent mounts from step 2 (e.g. `~/.config` gets
mounted ro, then `~/.config/nvim/` gets mounted rw on top).

## Environment

`--clearenv` strips everything.  Only these are re-injected:

```
HOME PATH USER LOGNAME SHELL TERM LANG LC_ALL LC_CTYPE LC_COLLATE
EDITOR VISUAL PAGER BROWSER COLORTERM NO_COLOR CLICOLOR
XDG_CACHE_HOME XDG_CONFIG_HOME XDG_DATA_HOME XDG_STATE_HOME
JAVA_HOME GOPATH GOROOT CARGO_HOME RUSTUP_HOME NVM_DIR
PYTHONPATH PERL5LIB ANDROID_HOME ANDROID_SDK_ROOT
GRADLE_HOME CUDA_PATH VCPKG_ROOT MAKEFLAGS
```

Variables starting with `SANDBOX_` always pass through regardless.

Defined as `_SAFE_ENV_VARS` in the script.  Secrets like
`DEEPSEEK_API_KEY`, `SSH_AUTH_SOCK`, `SSLKEYLOGFILE`, `DISPLAY`,
`DBUS_SESSION_BUS_ADDRESS` are never passed.

## Network

Isolated by default (`--unshare-net`).  Opt-in with `--net` flag or
`SANDBOX_NET=1`.  When network is enabled, the namespace is shared
with the host — no firewall rules or further restrictions apply.

## Container sockets

Not mounted by default.  Opt-in with flags:

| Flag | Default socket | Notes |
|------|---------------|-------|
| `--docker` | `/var/run/docker.sock` | Respects `DOCKER_HOST=unix://...`; TCP daemons need `--net` |
| `--podman` | `/run/user/$UID/podman/podman.sock` | |
| `--containerd` | `/run/containerd/containerd.sock` | |

Sockets are bind-mounted read-write (`--bind-try`).  Missing sockets are
a silent no-op.

## Extra mounts (`--add`)

`--add ro|rw SRC DST` mounts arbitrary paths into the sandbox.

- `ro` — read-only bind mount (`--ro-bind`)
- `rw` — read-write bind mount (`--bind`)
- Both SRC and DST are required.  To mount at the same path, repeat it:
  `--add rw /tmp/scratch /tmp/scratch`.
- Fails immediately if SRC doesn't exist (unlike socket flags which
  use `--bind-try`).
- Repeatable for multiple paths.

## Namespaces

| Namespace | Flag |
|-----------|------|
| User | `--unshare-user` |
| PID | `--unshare-pid` |
| IPC | `--unshare-ipc` |
| UTS | `--unshare-uts` |
| Cgroup | `--unshare-cgroup-try` |
| Network | `--unshare-net` (unless `--net`) |

`--die-with-parent` tears down when the parent exits.

## /etc/hosts

Overlaid with `--ro-bind` from a tempfile:

```
127.0.0.1   localhost
::1         localhost ip6-localhost ip6-loopback
```

Prevents leaking internal hostnames from the real `/etc/hosts`.
Uses a tempfile rather than `--ro-bind-data` because bwrap writes
the data inside the destination directory, which is already a
read-only mount.

## Seccomp

Optional.  If `seccomp.bpf` exists next to the script, it's loaded with
`--seccomp`.  Not enforced by default.

## Dependencies

- bubblewrap ≥ 0.11 (for `--bind-data`, `--clearenv`, `--seccomp`)
- Python 3.10+

## Files

```
~/.config/sandbox-here/
├── sandbox-here
├── seccomp.bpf        (optional)
├── AGENTS.md
└── README.md
```

Wrapper (in `~/.zshrc`):

```zsh
sbh() { "$HOME/.config/sandbox-here/sandbox-here" "$@"; }
```

## Invocation from an agent harness

The agent (pi, codex, etc.) typically invokes `sbh` like:

```bash
sbh pi
# or with network:
SANDBOX_NET=1 sbh pi
```

The agent's own config (`~/.pi/`) is discovered and mounted read-write.
Other sensitive paths (`.ssh`, `.codex`, etc.) are invisible.

Session logs and conversation history write to `~/.pi/agent/sessions/`,
which persists because `~/.pi/` is a `--bind-try` to the real disk.
Everything else in `$HOME` is tmpfs and disappears on exit.
