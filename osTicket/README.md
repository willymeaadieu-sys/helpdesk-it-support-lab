# Ticketing System — osTicket Setup

Part of the [Help Desk & IT Support Lab] portfolio project.

This section covers standing up a self-hosted support ticketing system from a bare Ubuntu Server install — LAMP stack provisioning, database and user creation, file transfer across an isolated VM network, and the osTicket web installer. This environment will be used to simulate and document realistic Tier 1 support tickets.

## Contents

- [Environment Overview](#environment-overview)
- [1. Server Provisioning](#1-server-provisioning)
- [2. Apache Installation](#2-apache-installation)
- [3. MySQL Installation & Hardening](#3-mysql-installation--hardening)
- [4. Database & User Creation](#4-database--user-creation)
- [5. PHP Installation](#5-php-installation)
- [6. File Transfer — Host to VM](#6-file-transfer--host-to-vm)
- [7. Deploying osTicket to the Web Root](#7-deploying-osticket-to-the-web-root)
- [8. Config File & Web Installer](#8-config-file--web-installer)
- [9. Post-Install Cleanup](#9-post-install-cleanup)
- [Troubleshooting Log](#troubleshooting-log)

---

## Environment Overview

| Component | Detail |
|---|---|
| Hypervisor | Oracle VirtualBox |
| VM Name | osTicket-Server |
| OS | Ubuntu Server 26.04 LTS |
| Network Mode | NAT (internet access required for package installs) |
| Guest IP | `10.0.2.15` (DHCP via NAT) |
| SSH User | `willy` |
| Web Stack | Apache 2.4.66, PHP 8.5.4, MySQL 8.4.10 |
| Application | osTicket v1.17.8 |
| Host → Guest SSH | Port forward `2222` → `22` |
| Host → Guest HTTP | Port forward `8080` → `80` |

---

## 1. Server Provisioning

Installed Ubuntu Server 26.04 LTS on a new VM. Network adapter set to **NAT**, not Internal Network — unlike the AD lab's domain controller, this VM needs outbound internet access to pull packages from Ubuntu's repositories.

**Troubleshooting note:** initial Ubuntu Desktop ISO download was corrupted (0-byte file). Resolved by re-downloading the correct Server ISO.

OpenSSH server was installed during setup to allow remote administration from the host machine.

---

## 2. Apache Installation

```bash
sudo apt update
sudo apt install apache2 -y
```

Verified the service was active and enabled to start on boot:

```bash
sudo systemctl status apache2
```

<img width="1532" height="830" alt="01-apache-installed" src="https://github.com/user-attachments/assets/78716a4f-90e1-4645-8618-a8b1e8738a2a" />

Confirmed with:

```
Active: active (running)
```

**Troubleshooting note:** Apache logged `AH00558: Could not reliably determine the server's fully qualified domain name` on startup. This is a benign warning — Apache falls back to a default when it can't resolve its own FQDN on a NAT/DHCP network — and does not affect functionality. Optionally resolved by explicitly setting `ServerName` in a new config file under `/etc/apache2/conf-available/`.

---

## 3. MySQL Installation & Hardening

```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation
```

Configuration selected during hardening:

| Prompt | Choice |
|---|---|
| Password validation policy | MEDIUM (length ≥ 8, mixed case, numbers, special characters) |
| Remove anonymous users | Yes |
| Disallow remote root login | Yes |
| Remove test database | Yes |
| Reload privilege tables | Yes |

<img width="1538" height="1218" alt="02-mysql-secure-install-complete" src="https://github.com/user-attachments/assets/deec789b-701b-4877-9aa5-192fa5e6f497" />

**Note on root authentication:** MySQL 8 on Ubuntu uses `auth_socket` authentication for `root@localhost` by default rather than a traditional password. This means root access requires `sudo mysql` (authenticating via the Linux OS user) rather than `mysql -u root -p`. This is expected behavior, not a misconfiguration.

---

## 4. Database & User Creation

Logged in as root via `sudo mysql` and created a dedicated, least-privilege database user for the application — following the same principle used in the AD lab's shared-folder permissions (scope access only to what's needed):

```sql
CREATE DATABASE osticket CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'osticket_user'@'localhost' IDENTIFIED BY '********';

GRANT ALL PRIVILEGES ON osticket.* TO 'osticket_user'@'localhost';

FLUSH PRIVILEGES;
```

<img width="1538" height="1214" alt="03-mysql-database-user-created" src="https://github.com/user-attachments/assets/3036a12c-3b52-49f5-9699-1696f4208345" />

- `'osticket_user'@'localhost'` restricts the account to local connections only — the application and database live on the same host, so no remote DB access is needed.
- `ON osticket.*` scopes privileges to only the `osticket` database, not the entire server.

---

## 5. PHP Installation

```bash
sudo apt install php php-cli php-mysql php-gd php-mbstring php-curl php-xml php-zip php-intl php-bcmath -y
```

<img width="1536" height="1204" alt="04-php-installed" src="https://github.com/user-attachments/assets/4b2fff7c-bb99-4342-82e3-8cdbd3be5ff6" />

**Troubleshooting note:** `php-imap` was requested initially but returned `Package 'php-imap' has no installation candidate`.

!<img width="1538" height="1214" alt="05-php-imap-unavailable" src="https://github.com/user-attachments/assets/61e2b07b-18b6-4adb-be18-6d3e81e030a0" />

A repository search confirmed no `php-imap` (or version-specific equivalent) package was available for this release. Since `php-imap` only supports osTicket's "fetch tickets from an email inbox" feature — not required for this lab, where tickets are created through the web form — it was omitted from the install without impacting core functionality.

Verified with:

```bash
php -v
```

```
PHP 8.5.4 (cli) (built: May 25 2026)
```

**Note:** Installing `libapache2-mod-php` automatically switched Apache's MPM (multi-processing module) from `mpm_event` to `mpm_prefork`, since traditional `mod_php` is incompatible with the event-based MPM. This is expected behavior any time PHP is installed as an Apache module.

---

## 6. File Transfer — Host to VM

The osTicket package was downloaded on the Windows host and needed to be transferred into the VM. Since the VM uses NAT networking, inbound connections (like SCP) are blocked by default.

**Solution:** added a VirtualBox NAT port-forwarding rule:

| Name | Protocol | Host Port | Guest Port |
|---|---|---|---|
| SSH | TCP | 2222 | 22 |

<img width="1008" height="688" alt="06-nat-port-forwarding-ssh" src="https://github.com/user-attachments/assets/7cd11e53-3a84-4a10-8baa-100af84cbb46" />

Transferred the extracted osTicket folder from the Windows host via SCP, targeting the loopback address and forwarded port:

```powershell
scp -P 2222 -r "C:\Users\willy\Downloads\osTicket-v1.17.8" willy@127.0.0.1:/home/willy/
```

<img width="2052" height="1214" alt="07-scp-file-transfer" src="https://github.com/user-attachments/assets/e79c6232-389f-448c-a633-1f58e74e0111" />

**Troubleshooting note:** an early connection attempt failed immediately after the SSH host-key prompt with `Connection closed`. Investigation of `/var/log/auth.log` showed the disconnect occurred `[preauth]` — before authentication completed. `PasswordAuthentication` was confirmed enabled in both `sshd_config` and `sshd_config.d/50-cloud-init.conf`, ruling out a config issue. The failure did not reproduce on retry; the transfer completed successfully on the second attempt.

---

## 7. Deploying osTicket to the Web Root

osTicket's extracted package includes non-web files (docs, setup scripts) alongside the actual application root in `upload/`. Only the contents of `upload/` are copied into Apache's web root:

```bash
sudo cp -r ~/osTicket-v1.17.8/upload/* /var/www/html/
sudo chown -R www-data:www-data /var/www/html/
sudo find /var/www/html/ -type d -exec chmod 755 {} \;
sudo find /var/www/html/ -type f -exec chmod 644 {} \;
```

**Troubleshooting note:** a typo (`chomd` instead of `chmod`) caused `find` to fail once per file in the target directory:

<img width="1548" height="1224" alt="08-chmod-typo-troubleshooting" src="https://github.com/user-attachments/assets/269bebc4-0789-4a05-b087-f54c6425428d" />

No changes were applied on the failed attempt, so the corrected command was simply re-run with no side effects.

A second issue arose after copying the files: Apache's default `index.html` (left over from the initial Apache install) was still present in `/var/www/html/` and took priority over osTicket's `index.php`, causing the Apache default placeholder page to load instead of the osTicket installer:

<img width="2682" height="1780" alt="09-apache-default-page-issue" src="https://github.com/user-attachments/assets/cb4ca228-7b1e-4db5-a9f1-3286e85e35b2" />

Resolved by removing the leftover file:

```bash
sudo rm /var/www/html/index.html
```

---

## 8. Config File & Web Installer

osTicket ships a sample config file rather than a live one, to prevent a fresh install from referencing database credentials that don't exist yet. The install wizard writes real credentials into this file, which requires it to be temporarily writable by the web server:

```bash
sudo cp /var/www/html/include/ost-sampleconfig.php /var/www/html/include/ost-config.php
sudo chmod 666 /var/www/html/include/ost-config.php
```

Added a second NAT port-forwarding rule to reach the web installer from the host browser:

| Name | Protocol | Host Port | Guest Port |
|---|---|---|---|
| HTTP | TCP | 8080 | 80 |

With `index.html` removed and the port forward in place, the osTicket installer loaded successfully, with all required prerequisites passing:

<img width="2662" height="1792" alt="10-osticket-installer-prerequisites" src="https://github.com/user-attachments/assets/bdfe9f91-0613-42df-bb1b-2673aade8afe" />

Completed the install wizard at `http://127.0.0.1:8080` with the following settings:

| Field | Value |
|---|---|
| MySQL Hostname | `localhost` |
| MySQL Database | `osticket` |
| MySQL Username | `osticket_user` |
| MySQL Table Prefix | `ost_` |

**Troubleshooting note:** the installer rejected `admin` as an administrator username with a "Bad username" validation error — osTicket blocks common/default usernames as a basic security measure:

<img width="1020" height="72" alt="11-bad-username-validation" src="https://github.com/user-attachments/assets/6e9e7b8c-7e88-4b9e-8eb3-033b8a1cbaf3" />

Resolved by choosing a more specific username instead.

Installation completed successfully:

<img width="1330" height="1782" alt="12-installation-congratulations" src="https://github.com/user-attachments/assets/2e671d70-2fd8-4a9d-9f3d-23f2be0d62d1" />
---

## 9. Post-Install Cleanup

Per the installer's own completion instructions, the config file's permissions were locked back down immediately after install to remove the temporary world-writable access:

```bash
sudo chmod 0644 /var/www/html/include/ost-config.php
```

A config file left world-writable after installation would allow any local user or process to read or modify live database credentials — a significant local privilege-escalation risk. This mirrors the same least-privilege principle applied throughout the AD lab (e.g., restricting share/NTFS permissions to specific security groups rather than leaving broad default access in place).

**Access points:**

| Interface | URL |
|---|---|
| Customer Portal | `http://127.0.0.1:8080/` |
| Staff Control Panel | `http://127.0.0.1:8080/scp` |

Logged into the Staff Control Panel to confirm the full stack — Apache, PHP, and MySQL — was working end-to-end:

<img width="2678" height="1790" alt="13-staff-control-panel-dashboard" src="https://github.com/user-attachments/assets/cab9ace1-5b11-4c38-8218-3ac987e2e2b9" />
---

## Troubleshooting Log

| Issue | Root Cause | Resolution |
|---|---|---|
| `php-imap` install failure | Package not available in this Ubuntu release's repos | Omitted; non-essential for this lab's use case |
| SCP connection closed `[preauth]` | Undetermined — did not reproduce on retry | Retried transfer, succeeded |
| `chmod` typo (`chomd`) | Typing error | Caught via `find` error output, corrected, re-ran with no side effects |
| Apache serving default page instead of osTicket | Leftover `index.html` in web root took priority over `index.php` | Removed `index.html` |
| Installer rejected `admin` as username | osTicket blocks common default usernames | Chose a more specific username |
| VirtualBox VM console unresponsive to keyboard input | VirtualBox "Keyboard failure" — input capture issue | Logged in via SSH instead of the VM console |
