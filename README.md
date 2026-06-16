# sandbox-here

Run a command so it can only see the current directory and its own config
files.  Everything else in your home directory is either read-only or
hidden completely.

It's a Python script that builds a bubblewrap container and drops your
command inside.  No SUID binaries, no daemon, no root.  Just Linux
namespaces.

## Why?

AI coding agents run LLM-generated shell commands on your machine.  Even
with human approval on each one, there's a lot that can go wrong — reading
SSH keys, scraping shell history, exfiltrating `~/Documents`, modifying
git config, leaking internal hostnames.

`sandbox-here` closes that gap.  The command sees the project directory
and its own config.  That's it.

## What it's not

It's not a jail for untrusted code.  A determined attacker will get out.
User namespaces are a hardening tool, not a security boundary.  With
`--net`, the command can make arbitrary outbound connections.  There's no
seccomp filter unless you add one, no resource limits, and writes to the
app's own config directory persist on disk.

Think of it like locking your car.  Stops casual snooping and mistakes.
Won't stop someone with a brick.  For running code you genuinely don't
trust, use a VM or a separate machine.

## Installation

```bash
mkdir -p ~/.config/sandbox-here
cp sandbox-here ~/.config/sandbox-here/
chmod +x ~/.config/sandbox-here/sandbox-here
```

Needs bubblewrap 0.11+ and Python 3.10+.  That's it.

Then alias it in your shell:

```bash
alias sbh="$HOME/.config/sandbox-here/sandbox-here"
```

(`sbh` is what I use day-to-day.  The full name works too.)

## Usage

```bash
# Open a shell sandboxed to the current directory
sbh

# Run a command
sbh npm install
sbh make
sbh cargo build

# Allow network — for package managers, curl, etc.
sbh --net pip install requests
```

When you run `sbh npm`, the script figures out the app is `npm` and mounts
`~/.npm/` read-write so it can use the cache.  `sbh cargo` gets `~/.cargo/`,
`sbh git` gets `~/.gitconfig`, and so on.  If you run `sbh` with no command
it opens your shell and makes the shell's own configs writable.

Symlinks are followed.  If `/usr/bin/sh` points to `bash`, running `sbh sh`
grants write access to `~/.bashrc` and `~/.config/bash/`.

## What gets blocked

Sensitive paths are invisible — not even mounted read-only:

```
.ssh .gnupg .codex .pki .docker .mozilla .thunderbird .librewolf
.aws .azure .gcloud .gsutil .copilot .electrum .tor .gnome .kde4
```

Environment variables are wiped clean.  Only a short safelist gets
re-injected (`PATH`, `HOME`, `TERM`, `LANG`, a few toolchain vars).
`DEEPSEEK_API_KEY`, `SSH_AUTH_SOCK`, `SSLKEYLOGFILE`, `DISPLAY`,
`DBUS_SESSION_BUS_ADDRESS` — all stripped.  If you need to pass something
through, prefix it with `SANDBOX_`.

`/etc/hosts` is overlaid with a minimal file so your internal hostnames
(LAN machines, Tailnet nodes, etc.) don't leak.

Network is off by default.  `--net` turns it on.

## What doesn't get blocked

Any dotfile not on the blocklist is visible read-only — `.bashrc`,
`.gitconfig`, `.vimrc`, and so on.  If you stash secrets in those,
they're visible.

With `--net`, the command has the same network access you do.  It can hit
localhost services, scan your LAN, make outbound connections.

The app's own config directory is writable and persists to disk.  If you
run `sbh pi`, the agent's config at `~/.pi/` survives sandbox teardown.
That's by design (the agent needs it), but it means a compromised agent
can modify its own settings to persist across restarts.

`/etc/passwd`, `/proc/cpuinfo`, and other world-readable system files are
visible.  No attempt is made to hide them.

## Seccomp

Optional.  Drop a `seccomp.bpf` next to the script and it gets loaded.
bubblewrap's repo has example policies.  Without one, no syscall filtering
happens.

## Files

```
~/.config/sandbox-here/
├── sandbox-here      the script
├── seccomp.bpf       optional syscall filter
├── README.md         this file
└── AGENTS.md         reference for AI agents running inside the sandbox
```

## License

Do whatever you want.  It's a few hundred lines of Python that wraps
bubblewrap.  Don't care.
