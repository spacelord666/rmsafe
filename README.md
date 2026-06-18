# rmsafe — Safety Wrapper for Destructive Linux Commands

`rmsafe` intercepts dangerous operations (`rm -rf /`, `dd` to block devices,
`mkfs` on system partitions, etc.) and requires you to type a random
confirmation phrase before proceeding. This prevents catastrophic accidents
while allowing deliberate destructive operations.

## Installation

```bash
# Copy to a PATH directory
cp rmsafe /usr/local/bin/rmsafe
chmod +x /usr/local/bin/rmsafe

# Add automatic aliases to your shell config
echo 'eval "$(rmsafe --init)"' >> ~/.bashrc
# or for zsh:
echo 'eval "$(rmsafe --init)"' >> ~/.zshrc
```

## Usage

```
rmsafe [options] <command> [args...]
```

### Options

| Flag              | Description                            |
|-------------------|----------------------------------------|
| `--force`, `-f`   | Skip confirmation (for scripts)        |
| `--dry-run`, `-n` | Check without executing                |
| `--init`          | Print shell aliases for auto-protection |
| `--help`, `-h`    | Show help message                      |

### Examples

```bash
# Requires confirmation:
rmsafe rm -rf /etc
rmsafe rm -rf /boot
rmsafe rm -rf /*
rmsafe rm -rf /
rmsafe dd if=/dev/zero of=/dev/sda bs=4M
rmsafe mkfs.ext4 /dev/sda1
rmsafe chmod -R 0 /
rmsafe chown -R nobody /
rmsafe mv /etc /tmp
rmsafe find / -delete

# Passes through without confirmation (safe operations):
rmsafe rm -rf /var/log/*
rmsafe rm -rf /tmp/*
rmsafe rm -rf /var/cache/*
rmsafe rm -f /etc/passwd                    # no -r flag
rmsafe rm -rf /home/user/projects           # /home not in protected paths
```

## What Is Protected

By default, these paths are protected:

```
/  /etc  /boot  /lib  /lib32  /lib64  /usr  /bin
/sbin  /sys  /proc  /dev  /run  /root
```

A path is considered dangerous **only** when the command targets the directory
itself or all its contents (e.g., `/etc` or `/etc/*`). Targeting subdirectories
like `/etc/apt/sources.list.d/*` passes through without confirmation.

## Confirmation Prompt

When a dangerous operation is detected, `rmsafe` shows:

```
═══════════════════════════════════════════
  DANGEROUS OPERATION DETECTED
═══════════════════════════════════════════

  Command: rm -rf /etc

  This action could CRIPPLE or DESTROY your system!

  To proceed, type the following phrase exactly:

      inclinable-moxahala-funereal

  >
```

You must type the phrase **exactly** (case-sensitive). Wrong input aborts
the operation.

## Environment Variables

### `RMSAFE_CRITICAL_PATHS`
Override the list of protected paths (colon-separated).

```bash
export RMSAFE_CRITICAL_PATHS="/:/etc:/boot:/lib:/usr:/var"
```

### `RMSAFE_SKIP`
Regex pattern matching command names to always allow (bypass all checks).

```bash
export RMSAFE_SKIP="^apt-get$|^dpkg$|^pacman$"
```

## Command Reference: What Triggers a Check

| Command                  | Dangerous When                                         |
|--------------------------|--------------------------------------------------------|
| `rm`                     | `-r`/`-R` flag + target is a protected path or `/*`    |
| `dd`                     | `of=` points to `/dev/sd*`, `/dev/nvme*`, etc.         |
| `mkfs.*`, `mke2fs`, etc. | Argument is a block device path                         |
| `chmod`                  | `-R` flag + target is `/`                               |
| `chown`                  | `-R` flag + target is `/`                               |
| `mv`                     | Source is a protected path                              |
| `find`                   | Starts at a protected path + uses `-delete` or `-exec`  |
| Any other command        | Any non-flag argument resolves to a protected path      |
