# mariadb-galera

<img src="https://img.shields.io/badge/Version-1.1.0-blue?style=flat-square" alt="Version">  
<img src="https://img.shields.io/badge/MariaDB-11.8-orange?style=flat-square&logo=mariadb" alt="MariaDB 11.8">  
<img src="https://img.shields.io/badge/Galera-26.4-brightgreen?style=flat-square" alt="Galera 26.4">  
<img src="https://img.shields.io/badge/Ubuntu/Debian-supported-brightgreen?style=flat-square" alt="Ubuntu/Debian">  
<img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="MIT License">

**The friendliest way to set up a MariaDB Galera Cluster in one command.**

> A robust, extremely defensive **Bash** script that installs MariaDB 11.8 + Galera, manages cluster IP addresses in `/etc/mariadb-galera/ip.conf`, and guides you through bootstrap vs join setup.  
> Part of the Wilgat defensive tool family (aligned with [ciao](https://github.com/Wilgat/ciao)).

---

## ✨ Features

- One-liner install (`curl | sudo bash`) — **root-only**
- Installs official MariaDB 11.8 repository + Galera packages on Debian/Ubuntu
- Manages cluster member IPs via `/etc/mariadb-galera/ip.conf` (one IP per line)
- **Interactive guided setup** — detects external IP, lists local IPs, helps add/remove cluster members
- **Non-interactive install mode** (`install` subcommand)
- Asks whether this is the **first node** (bootstrap) or a joining node
- Generates basic Galera configuration (`/etc/mysql/conf.d/galera.cnf`)
- Self-installing, self-updating (`--self-update`), and version checking
- `--force` / `--reinstall`, `--quiet` support
- Extremely defensive coding style with repeated safe defaults

---

## 🚀 Quick Installation

**System-wide (root required):**
```bash
curl -fsSL https://raw.githubusercontent.com/Wilgat/mariadb-galera/main/mariadb-galera | sudo bash
```

After installation, run the interactive setup:
```bash
sudo mariadb-galera
```

---

## 📖 Usage

```bash
sudo mariadb-galera                    # Full interactive guided cluster setup (recommended)
sudo mariadb-galera install            # Non-interactive installation only
sudo mariadb-galera list               # List current cluster IPs
sudo mariadb-galera add <ip>           # Add an IP to the cluster
sudo mariadb-galera remove <ip>        # Remove an IP from the cluster
mariadb-galera version                 # Show current version
mariadb-galera version-check           # Compare with latest on GitHub
sudo mariadb-galera self-update        # Update to latest version
mariadb-galera help                    # Show detailed help
```

### Behavior
- **Interactive mode** (default): Detects IPs, manages `/etc/mariadb-galera/ip.conf`, generates config, and guides bootstrap vs normal start.
- **`install` subcommand**: Only installs MariaDB + Galera packages (no interactive setup).
- Root privileges are **required** for installation and configuration changes.

---

## Typical 3-Node Setup Example

This is the most common production setup for a MariaDB Galera Cluster.

### Node Information

| Node     | Hostname     | External IP     | Role                  |
|----------|--------------|-----------------|-----------------------|
| Node 1   | db-node-01   | 192.168.10.11   | First node (bootstrap)|
| Node 2   | db-node-02   | 192.168.10.12   | Joining node          |
| Node 3   | db-node-03   | 192.168.10.13   | Joining node          |

### Step-by-step on each node:

**On all three nodes (as root):**

1. Install the script:
   ```bash
   curl -fsSL https://raw.githubusercontent.com/Wilgat/mariadb-galera/main/mariadb-galera | sudo bash
   ```

2. Run the interactive setup:
   ```bash
   sudo mariadb-galera
   ```

3. During the interactive session:
   - The script will detect the current node's external IP
   - Add the current node's IP when prompted
   - Add the other two nodes' IPs
   - When asked **"Is this the FIRST computer in the cluster?"**
     - Answer **Yes** only on **Node 1**
     - Answer **No** on **Node 2** and **Node 3**

**Recommended order:**

- First run the full interactive setup on **Node 1** and choose **Yes** for first node (bootstrap).
- Then run the setup on **Node 2** and **Node 3**, choosing **No**.

After setup:
- Node 1: `sudo galera_new_cluster` (or `sudo systemctl start mariadb`)
- Node 2 & 3: `sudo systemctl start mariadb`

Check cluster status on any node:
```sql
SHOW STATUS LIKE 'wsrep_%';
```

---

## Important Platform Notes

### Supported Platforms
- **Ubuntu / Debian** — Excellent (primary target)
- **Other Debian-based** — Good

The script currently focuses on **apt-based** systems.

### Firewall Requirements
Galera Cluster requires these ports to be open **between all nodes**:

- `3306` — MariaDB client / SST
- `4567` — Galera replication (main)
- `4568` — Incremental State Transfer (IST)
- `4444` — State Snapshot Transfer (SST)

Example on Ubuntu/Debian:
```bash
sudo ufw allow 3306/tcp
sudo ufw allow 4567/tcp
sudo ufw allow 4568/tcp
sudo ufw allow 4444/tcp
```

---

## Platform Compatibility

| Platform                  | Status       | Notes |
|---------------------------|--------------|-------|
| **Ubuntu / Debian**       | Excellent    | Full support (apt) |
| **Alpine Linux**          | Not supported| No apt, different package manager |
| **RHEL / CentOS / Alma**  | Planned      | dnf/yum support coming later |
| **macOS**                 | Not supported| Not intended for production DB |
| **Git Bash (Windows)**    | Limited      | Installation may work, cluster not recommended |

---

## Program Structure (for curious people)

The script is intentionally kept **linear**, highly readable, and **extremely defensive** following the strict "ciao" coding style.

```text
┌─────────────────────────────────────────────────────────────┐
│  Header + Warnings + Project Constants                      │
│  (APP_NAME, VERSION, SCRIPT_URL, CONFIG_DIR, etc.)          │
├─────────────────────────────────────────────────────────────┤
│  Safe Variable Defaults (repeated on purpose)               │
│  Root Detection (IS_ROOT — mandatory for most operations)   │
│  Force Flags & Quiet Mode                                   │
├─────────────────────────────────────────────────────────────┤
│  Color Output + Logging Functions (die, info, warn, etc.)   │
├─────────────────────────────────────────────────────────────┤
│  Core Utility Functions                                     │
│   • is_installed()            ← robust install detection    │
│   • get_installed_version()                                 │
│   • version_check()                                         │
│   • self_update()                                           │
│   • perform_self_install()                                  │
├─────────────────────────────────────────────────────────────┤
│  Compatibility Functions (kept for CIAO style)              │
│   • in_path(), add_to_shell_path(), check_alpine_requirements() (debug only) │
├─────────────────────────────────────────────────────────────┤
│  MariaDB Galera Logic                                       │
│   • require_root()                                          │
│   • ensure_config_dir()                                     │
│   • detect_external_ip() + list_local_ips()                 │
│   • ip_conf_list() / add() / remove()                       │
│   • setup_mariadb_repo() + install_mariadb_galera()         │
│   • generate_galera_config()                                │
├─────────────────────────────────────────────────────────────┤
│  Help & Main Entry Point                                    │
│   • show_galera_help()                                      │
│   • main()                                                  │
│        ├── Argument parsing                                 │
│        ├── Special handlers (version, install, list, etc.)  │
│        ├── maybe_install() if needed                        │
│        └── Interactive / Non-interactive flow               │
└─────────────────────────────────────────────────────────────┘
```

---

## Why the Heavy Defensive Style?

This project strictly follows the **ciao defensive coding style**:
- Repeated safe defaults (`: "${VAR:=default}"`)
- Redundant root checks and error handling
- Heavy inline comments and `!!! DO NOT MODIFY OR SIMPLIFY !!!` blocks

**Purpose**:
- Survive harsh environments (`curl | sudo bash`, non-interactive shells, etc.)
- Protect against accidental "cleaning" by AI assistants or contributors
- Serve as a reliable template for other defensive tools in the Wilgat family

---

## Project Philosophy

> "Write code that is easy to copy, hard to break, and self-documenting."

---

## Contributing

Please respect the strict defensive coding style and protective comments when submitting changes.

---

## License

MIT

---

**Part of the Wilgat defensive tool family.**  
*Last updated: April 2026*

---

**Note**: MariaDB 11.8 is a current long-term supported (LTS) release. This tool provides a quick, reproducible way to set up a MariaDB Galera Cluster with managed IP configuration.

