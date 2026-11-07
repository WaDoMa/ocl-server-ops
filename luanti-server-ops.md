# Luanti Server Operations

Operational documentation for a Luanti 5.10.0 server running Mineclonia on Ubuntu in Oracle Cloud Infrastructure (OCI).

This document describes the current server configuration and procedures for administration, maintenance, backups, mod management, updates, and security.

## 1. System, User & Directory Architecture

### Server overview

| Component | Configuration |
|---|---|
| Operating system | Ubuntu on OCI |
| Architecture | aarch64 |
| Luanti version | 5.10.0 |
| Game | Mineclonia |
| Server service | `luanti-server.service` |
| Game port | UDP `30000` |
| Server configuration | `/etc/luanti/default.conf` |
| Instance directory | `/var/lib/luanti/default` |
| World directory | `/var/lib/luanti/default/worlds/mineclonia_world` |
| Game directory | `/var/lib/luanti/default/games/mineclonia` |
| Mods directory | `/var/lib/luanti/default/mods` |
| Log file | `/var/log/luanti/default/server.log` |

### Directory structure

```text
/etc/luanti/
└── default.conf

/var/lib/luanti/default/
├── games/
│   └── mineclonia/
├── mods/
│   ├── whitelist/
│   └── lib_chatcmdbuilder/
└── worlds/
    └── mineclonia_world/
        ├── world.mt
        ├── map.sqlite
        ├── players.sqlite
        ├── auth.sqlite
        └── mod_storage.sqlite

/var/log/luanti/default/
└── server.log
```

The server runs as a systemd service with:

```ini
DynamicUser=yes
```

It therefore does not rely on a permanently assigned Linux login account.

Systemd manages the service's state and log directories through:

```ini
StateDirectory=luanti/default
LogsDirectory=luanti/default
```

The instance data is located under `/var/lib/luanti/default`.

## 2. Systemd Service Configuration

Luanti is installed as a systemd service:

```text
/usr/lib/systemd/system/luanti-server.service
```

The packaged service uses `minetest_game` as its default game. We want to run Mineclonia instead.

To achieve this without modifying the original package service file, we create a systemd override.

### Create the systemd override

First, create the override directory if it does not already exist:

```bash
sudo mkdir -p /etc/systemd/system/luanti-server.service.d
```

Then create the override file:

```bash
sudo nano /etc/systemd/system/luanti-server.service.d/override.conf
```

Insert the following configuration:

```ini
[Service]
DynamicUser=yes
User=
Group=
ExecStart=
ExecStart=/usr/libexec/luanti/luantiserver --config /etc/luanti/default.conf --logfile /var/log/luanti/default/server.log --gameid mineclonia
```

Save the file and exit the editor.

The empty `User=`, `Group=` and `ExecStart=` assignments reset the corresponding settings from the packaged service definition. The following `ExecStart=` specifies the command to start the server with Mineclonia.

`DynamicUser=yes` is retained.

### Apply the override

Tell systemd to reload its configuration:

```bash
sudo systemctl daemon-reload
```

Restart the service:

```bash
sudo systemctl restart luanti-server.service
```

Check whether the service started successfully:

```bash
sudo systemctl status luanti-server.service
```

Inspect the effective service configuration:

```bash
sudo systemctl cat luanti-server.service
```

The output should show both the original packaged service and the local override.

### Service management

Start, stop, or restart the server:

```bash
sudo systemctl start luanti-server.service
sudo systemctl stop luanti-server.service
sudo systemctl restart luanti-server.service
```

Inspect recent journal output:

```bash
sudo journalctl -u luanti-server.service -n 100 --no-pager
```

Follow the server log:

```bash
sudo tail -f /var/log/luanti/default/server.log
```

## 3. Network & Firewall Configuration (OCI Specific)

The server listens on UDP port `30000`.

Two separate firewall layers must allow incoming traffic:

1. The OCI network security configuration.
2. The Ubuntu instance firewall.

### OCI Security List

The OCI ingress rules must allow:

| Protocol | Destination port | Source |
|---|---:|---|
| UDP | 30000 | `0.0.0.0/0` |
| TCP | 22 | Restricted SSH source as configured |

The OCI rule is necessary even if the local firewall permits the traffic.

### Instance firewall

The system uses `iptables` with the `iptables-nft` backend.

The persistent base rules are managed through:

```text
/etc/iptables/rules.v4
```

The firewall policy includes:

- Allow established and related connections.
- Allow loopback traffic.
- Allow SSH on TCP port `22`.
- Allow Luanti on UDP port `30000`.
- Reject other incoming traffic.
- Reject forwarded traffic.

Fail2ban adds its own dynamic chains:

```text
f2b-sshd
f2b-luanti-whitelist
```

The relevant traffic is sent through these chains before reaching the general allow rules.

**Important:** Fail2ban creates and manages its own chains dynamically. They are not part of the persistent base rules in `rules.v4`.

Do not repeatedly insert duplicate rules with `iptables -I INPUT`. The existing rule order is intentional.

Also preserve the OCI-related `InstanceServices` rules. They are required for OCI instance services and must not be removed when modifying the firewall.

### Inspect firewall rules

```bash
sudo iptables -L INPUT -n -v --line-numbers
sudo iptables -L f2b-sshd -n -v
sudo iptables -L f2b-luanti-whitelist -n -v
```

Inspect the persistent rules:

```bash
sudo cat /etc/iptables/rules.v4
```

## 4. Fail2ban Brute-Force Protection

Fail2ban protects two services:

| Jail | Purpose | Log source |
|---|---|---|
| `sshd` | SSH login protection | systemd journal |
| `luanti-whitelist` | Repeated attempts to join without whitelist authorization | Luanti server log |

### Luanti filter

File:

```text
/etc/fail2ban/filter.d/luanti-whitelist.conf
```

Active configuration:

```ini
[Definition]
failregex = Player with the name "[^"]+" tried to connect from (?:.*:)?<HOST> but it was disallowed
ignoreregex =
```

The filter matches log entries indicating that a player was denied because of the whitelist.

The regular expression has been tested against the server log using `fail2ban-regex`.

### Jail configuration

File:

```text
/etc/fail2ban/jail.local
```

Current configuration:

```ini
[DEFAULT]
#banaction = ufw
#banaction_allports = ufw
actionstart_on_demand = false
banaction = iptables-multiport
banaction_allports = iptables-allports

[sshd]
enabled = true
journalmatch = _SYSTEMD_UNIT=ssh.service + _COMM=sshd + _COMM=sshd-session
action = iptables-multiport[name=sshd, port=ssh, protocol=tcp, actionstart_on_demand=false]

[luanti-whitelist]
enabled = true
filter = luanti-whitelist
logpath = /var/log/luanti/default/server.log
backend = polling
port = 30000
protocol = udp
maxretry = 5
findtime = 10m
bantime = 1h
action = iptables-multiport[name=luanti-whitelist, port=30000, protocol=udp, actionstart_on_demand=false]
```

The Luanti jail uses the following limits:

| Setting | Value |
|---|---:|
| Maximum failed attempts | 5 |
| Detection window | 10 minutes |
| Ban duration | 1 hour |
| Protocol | UDP |
| Port | 30000 |
| Log backend | polling |

The explicit `action` definitions with `actionstart_on_demand=false` are important for this setup. They ensure that the corresponding iptables chains and jumps are created when the jails start.

Do not simplify these action definitions to a generic `banaction` setting without verifying that the chains are still created correctly.

### Fail2ban administration

Check overall status:

```bash
sudo fail2ban-client status
```

Inspect an individual jail:

```bash
sudo fail2ban-client status sshd
sudo fail2ban-client status luanti-whitelist
```

Reload configuration:

```bash
sudo systemctl restart fail2ban
```

Manually ban an IP:

```bash
sudo fail2ban-client set luanti-whitelist banip <IP_ADDRESS>
```

Manually unban an IP:

```bash
sudo fail2ban-client set luanti-whitelist unbanip <IP_ADDRESS>
```

Test the Luanti filter against the log:

```bash
sudo fail2ban-regex \
  /var/log/luanti/default/server.log \
  /etc/fail2ban/filter.d/luanti-whitelist.conf
```

The filter has been verified against historical log entries. Automatic banning after five new matching events is configured, but has not been independently confirmed as an end-to-end test.

## 5. Game & Mod Installation

### Mineclonia

Mineclonia is installed in:

```text
/var/lib/luanti/default/games/mineclonia
```

Official repository:

```text
https://codeberg.org/Mineclonia/Mineclonia
```

Local branch:

```text
main
```

Remote:

```text
origin
```

The server selects Mineclonia through the systemd override:

```text
--gameid mineclonia
```

The world also identifies the game:

```ini
gameid = mineclonia
```

The installed checkout has been verified against the official `origin/main` branch. A `fetch` followed by a status check showed no commits missing at the time of verification.

### Whitelist mod

The Whitelist mod is installed at:

```text
/var/lib/luanti/default/mods/whitelist
```

Repository:

```text
https://gitlab.com/zughy-friends-minetest/whitelist.git
```

The installed checkout uses the `master` branch.

### ChatCmdBuilder dependency

The Whitelist mod depends on `lib_chatcmdbuilder`.

Official repository:

```text
https://github.com/rubenwardy/ChatCmdBuilder.git
```

Installation directory:

```text
/var/lib/luanti/default/mods/lib_chatcmdbuilder
```

Install it with:

```bash
sudo git clone \
  https://github.com/rubenwardy/ChatCmdBuilder.git \
  /var/lib/luanti/default/mods/lib_chatcmdbuilder
```

The dependency is enabled in the world configuration:

```ini
load_mod_lib_chatcmdbuilder = true
```

The installed checkout's current branch and tracking branch should be checked before updating. Do not assume its branch name.

## 6. Configuration & Whitelist Management

### Server configuration

File:

```text
/etc/luanti/default.conf
```

Current relevant settings:

```ini
server_name = YourLuantiServer
server_description = A Luanti server
server_announce = false
ipv6_server = true
motd = Welcome to Luanti. Enjoy your stay!

enable_pvp = true

default_game = mineclonia
map-dir = /var/lib/luanti/default/worlds/mineclonia_world

only_allow_trusted_logins = true
load_mod_whitelist = true
enable_whitelist = true
```

The whitelist settings also appear in `world.mt`. Keep the effective configuration consistent when changing them.

`only_allow_trusted_logins` is a separate Luanti server setting. It should not be confused with the Whitelist mod.

### World mod configuration

File:

```text
/var/lib/luanti/default/worlds/mineclonia_world/world.mt
```

Relevant settings:

```ini
gameid = mineclonia

enable_whitelist = true
load_mod_whitelist = true
load_mod_lib_chatcmdbuilder = true
```

The world uses SQLite for its storage backends:

```ini
mod_storage_backend = sqlite3
auth_backend = sqlite3
player_backend = sqlite3
backend = sqlite3
```

Other world settings:

```ini
enable_damage = true
creative_mode = false
world_name = mineclonia_world
```

### Whitelist commands

The available commands are:

```text
/whitelist on
/whitelist off
/whitelist who
/whitelist add <player>
/whitelist remove <player>
```

Examples:

Enable whitelist enforcement:

```text
/whitelist on
```

Disable whitelist enforcement:

```text
/whitelist off
```

Show the whitelist:

```text
/whitelist who
```

Add a player:

```text
/whitelist add PlayerName
```

Remove a player:

```text
/whitelist remove PlayerName
```

These are the commands reported by the installed mod's `/help whitelist` output.

### Administrator privileges

The administrator account used on this server is:

```text
AdminPlayer
```

It has been granted all privileges:

```text
/grant AdminPlayer all
```

Use an account with the necessary privileges to administer the whitelist.

### Inspect whitelist storage

The mod stores its data in the world's SQLite mod storage database:

```text
/var/lib/luanti/default/worlds/mineclonia_world/mod_storage.sqlite
```

Inspect the stored whitelist data:

```bash
sudo sqlite3 \
  /var/lib/luanti/default/worlds/mineclonia_world/mod_storage.sqlite \
  "SELECT CAST(key AS TEXT), CAST(value AS TEXT) FROM entries WHERE modname='whitelist';"
```

This is an inspection command. Do not edit the database directly while the server is running.

## 7. Maintenance Workflows & Updates

### General update procedure

For changes to Mineclonia or installed mods:

1. Check the current Git state.
2. Fetch remote changes.
3. Review available updates.
4. Stop the server.
5. Back up the world.
6. Pull the desired changes.
7. Start the server.
8. Check the service and log.

The `fetch` command only downloads repository information. It does not change the installed files.

The `pull` command updates the working tree.

### Git ownership and DynamicUser

The Luanti service uses `DynamicUser=yes`. Git may therefore report:

```text
fatal: detected dubious ownership in repository
```

The effective path used by Git can be under:

```text
/var/lib/private/luanti/default/
```

Use a command-specific `safe.directory` setting instead of adding a global exception:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/games/mineclonia \
  -C /var/lib/luanti/default/games/mineclonia \
  status -sb
```

For another repository, replace both paths with the corresponding repository directory.

This setting applies only to the individual Git invocation. It does not modify the global Git configuration.

### Update Mineclonia

Repository:

```text
https://codeberg.org/Mineclonia/Mineclonia
```

Branch:

```text
main
```

Fetch remote changes:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/games/mineclonia \
  -C /var/lib/luanti/default/games/mineclonia \
  fetch origin
```

Check the local branch and tracking information:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/games/mineclonia \
  -C /var/lib/luanti/default/games/mineclonia \
  branch -vv
```

Check the working tree:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/games/mineclonia \
  -C /var/lib/luanti/default/games/mineclonia \
  status -sb
```

List commits available remotely but not installed locally:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/games/mineclonia \
  -C /var/lib/luanti/default/games/mineclonia \
  log --oneline HEAD..origin/main
```

If the working tree is clean and the update is intended, update to `origin/main`:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/games/mineclonia \
  -C /var/lib/luanti/default/games/mineclonia \
  pull --ff-only origin main
```

`--ff-only` prevents Git from creating an automatic merge commit.

### Update the Whitelist mod

Repository:

```text
https://gitlab.com/zughy-friends-minetest/whitelist.git
```

The installed checkout uses `master`.

Fetch:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/mods/whitelist \
  -C /var/lib/luanti/default/mods/whitelist \
  fetch origin
```

Check status:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/mods/whitelist \
  -C /var/lib/luanti/default/mods/whitelist \
  status -sb
```

Review available commits:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/mods/whitelist \
  -C /var/lib/luanti/default/mods/whitelist \
  branch -vv
```

If the working tree is clean and the local branch tracks `origin/master`, update:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/mods/whitelist \
  -C /var/lib/luanti/default/mods/whitelist \
  pull --ff-only origin master
```

### Update lib_chatcmdbuilder

Repository:

```text
https://github.com/rubenwardy/ChatCmdBuilder.git
```

Installation directory:

```text
/var/lib/luanti/default/mods/lib_chatcmdbuilder
```

Fetch:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/mods/lib_chatcmdbuilder \
  -C /var/lib/luanti/default/mods/lib_chatcmdbuilder \
  fetch origin
```

Check the current branch and its tracking branch:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/mods/lib_chatcmdbuilder \
  -C /var/lib/luanti/default/mods/lib_chatcmdbuilder \
  branch -vv
```

Check the working tree:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/mods/lib_chatcmdbuilder \
  -C /var/lib/luanti/default/mods/lib_chatcmdbuilder \
  status -sb
```

If the working tree is clean and the branch tracks the intended remote branch, update with:

```bash
sudo git \
  -c safe.directory=/var/lib/private/luanti/default/mods/lib_chatcmdbuilder \
  -C /var/lib/luanti/default/mods/lib_chatcmdbuilder \
  pull --ff-only
```

This uses the configured upstream branch. The branch name should not be assumed without checking `branch -vv`.

### Back up the world

A consistent backup should be made while the server is stopped.

Stop the service:

```bash
sudo systemctl stop luanti-server.service
```

Confirm that it has stopped:

```bash
sudo systemctl status luanti-server.service
```

Create a world backup:

```bash
sudo tar -czf \
  /root/mineclonia-world-$(date +%F-%H%M%S).tar.gz \
  -C /var/lib/luanti/default/worlds \
  mineclonia_world
```

Back up the configuration separately:

```bash
sudo tar -czf \
  /root/luanti-config-$(date +%F-%H%M%S).tar.gz \
  /etc/luanti \
  /etc/fail2ban
```

The world backup contains the map, player data, authentication data, and mod storage.

The configuration backup contains the server and Fail2ban settings.

Store backups somewhere other than the server's only data disk if they are intended to protect against disk or instance loss.

### Restart after updates

After changing game or mod files:

```bash
sudo systemctl start luanti-server.service
```

Check the service:

```bash
sudo systemctl status luanti-server.service
```

Inspect the log for startup errors:

```bash
sudo tail -n 100 /var/log/luanti/default/server.log
```

### General troubleshooting checklist

If players cannot connect:

1. Check that `luanti-server.service` is running.
2. Check the server log for startup or connection errors.
3. Verify that UDP port `30000` is allowed in the OCI Security List.
4. Verify that the local INPUT chain allows UDP port `30000`.
5. Check the Fail2ban jail and its ban list.
6. Check that the whitelist is enabled and that the player is authorized.

Useful commands:

```bash
sudo systemctl status luanti-server.service
sudo ss -ulnp | grep 30000
sudo fail2ban-client status luanti-whitelist
sudo iptables -L INPUT -n -v --line-numbers
sudo tail -n 100 /var/log/luanti/default/server.log
```

If the server starts but a mod does not load, check:

- Whether the mod directory exists.
- Whether its dependencies are installed.
- Whether the required `load_mod_...` entry exists in `world.mt`.
- Whether the server log reports a Lua or dependency error.

---

## Operational Notes

- The server uses `DynamicUser=yes`; do not assume a permanent Linux service user.
- The active firewall backend is `iptables-nft`.
- Fail2ban manages its own dynamic iptables chains.
- Preserve OCI `InstanceServices` firewall rules.
- Stop the server before backing up or directly modifying world databases.
- The Whitelist mod and its ChatCmdBuilder dependency are separate mods.
- Mineclonia uses the official Codeberg repository and the `main` branch.
- The Whitelist mod uses the GitLab repository and the `master` branch.
- Check the configured upstream branch of ChatCmdBuilder before updating.
- Always inspect Git status before pulling changes.