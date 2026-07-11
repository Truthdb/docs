# Installing TruthDB

There are two ways to install TruthDB:

1. **As a service on an existing Debian/Ubuntu host** — via `apt` (recommended for most users).
2. **As a turnkey appliance on bare metal** — via the installer ISO that lays down Debian + TruthDB on a fresh machine.

---

## Install via apt (Debian / Ubuntu)

Supported distributions, amd64 only:

- Debian 12 (bookworm)
- Debian 13 (trixie)
- Ubuntu 22.04 (jammy)
- Ubuntu 24.04 (noble)

### 1. Add the apt repository

```sh
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://truthdb.github.io/apt/pubkey.asc \
  | sudo gpg --dearmor -o /etc/apt/keyrings/truthdb-archive-keyring.gpg
sudo chmod a+r /etc/apt/keyrings/truthdb-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/truthdb-archive-keyring.gpg] \
https://truthdb.github.io/apt $(. /etc/os-release && echo $VERSION_CODENAME) main" \
  | sudo tee /etc/apt/sources.list.d/truthdb.list

sudo apt-get update
```

### 2. Install

```sh
sudo apt-get install truthdb-server truthdb-cli
```

The service starts automatically and listens on TCP port **9623**.

### 3. Verify

```sh
systemctl status truthdb
truthdb-cli --help
```

### What got installed

| Path | Purpose |
|---|---|
| `/usr/bin/truthdb` | the database daemon |
| `/usr/bin/truthdb-cli` | the client |
| `/lib/systemd/system/truthdb.service` | systemd unit (enabled at install) |
| `/etc/truthdb/truthdb.toml` | configuration (conffile — your edits survive package upgrades) |
| `/var/lib/truthdb/` | data directory, owned by the `truthdb` system user |

### Configure

Edit `/etc/truthdb/truthdb.toml`. All values shown there are the embedded defaults; uncomment a line to override. Then:

```sh
sudo systemctl restart truthdb
```

### Diagnostics

```sh
systemctl status truthdb
journalctl -u truthdb -n 100
```

### Upgrade

```sh
sudo apt-get update
sudo apt-get upgrade truthdb-server truthdb-cli
```

If you have edited `/etc/truthdb/truthdb.toml`, `dpkg` will prompt before overwriting it. The default is to keep your version.

### Uninstall

```sh
sudo apt-get remove truthdb-server truthdb-cli       # keeps /etc/truthdb and /var/lib/truthdb
sudo apt-get purge truthdb-server truthdb-cli        # removes config, data, and the truthdb user
```

---

## Install via the bare-metal ISO

For dedicated appliances on a fresh machine: see [`truthdb-installer-iso`](https://github.com/Truthdb/installer-iso). Boot the ISO; it partitions the disk, installs Debian, and pre-installs TruthDB. The end state is identical to a Debian host that ran `apt-get install truthdb-server`.

---

## Troubleshooting

**`apt-get update` fails with `NO_PUBKEY` or signature errors.**
The keyring step did not complete. Re-run step 1 of the apt install above.

**`apt-get install truthdb-server` fails on Ubuntu 22.04 with a `libc6` version error.**
This should not happen — the packages are built against glibc 2.35 to match Ubuntu 22.04. If you see this, please open an issue with the exact `apt-get` output.

**Service does not start after install.**
Check `journalctl -u truthdb -n 100`. If a setting you changed in `/etc/truthdb/truthdb.toml` does not appear to take effect, verify the file is valid TOML — a parse error silently falls back to the embedded defaults. Restore the shipped file by reinstalling: `sudo apt-get install --reinstall truthdb-server` (this will prompt about the modified conffile).
