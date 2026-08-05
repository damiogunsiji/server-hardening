# Hardening a Fresh Linux VPS

Lock down a new Debian/Ubuntu VPS against bots and brute-force attacks.

---

## Prerequisites

SSH in as root and get the system current.

```bash
ssh root@your-server-ip
```

```bash
apt update && apt upgrade -y
```

Not all VPS images ship with ufw installed. Install it if missing:

```bash
apt install ufw -y
```

```bash
reboot
```

---

## 1. UFW Firewall

UFW is system-wide — once enabled, it protects all users and processes on the machine. No need for each user to configure their own firewall.

Allow only the ports you need. Replace `22` with your custom SSH port if you change it later.

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

Verify:

```bash
ufw status verbose
```

**Note:** Some VPS providers (e.g., Hetzner) offer a dashboard firewall that blocks ports externally. If you already allow only 22, 80, 443 there, UFW on the server is optional — you can skip this section.

Running both has **no negative effect** — they operate at different layers and don't conflict. The dashboard firewall blocks traffic before it reaches your server; UFW adds a second layer inside the OS. The only tradeoff is managing rules in two places.

---

## 2. Users

Running everything as root is risky. If a process or app gets compromised under root, the attacker owns your entire server. Different tasks need different privilege levels, so create separate users for each job.

### 2a. Create a Sudo User (Admin Access)

A sudo user is for day-to-day administration: installing packages, editing configs, restarting services. You use `sudo` to elevate only when needed, so mistakes or exploits don't automatically have root privileges.

```bash
adduser <username>   # you'll be prompted to set a password
usermod -aG sudo <username>
```

Test that you can SSH in as this user. Open a **new terminal** on your local machine:

```bash
ssh <username>@your-server-ip
```

If it works, continue. If it fails, fix the issue before moving on — this is your fallback after root login is disabled later.

Back in the root session, verify sudo works:

```bash
su - <username>
sudo whoami   # should print "root"
exit          # back to root
```

#### Create a Non-Root User with Passwordless Sudo Access

For automation (CI/CD, Ansible, scripts) or key-only setups where typing a sudo password isn't practical, create a user with no login password but full passwordless `sudo`:

```bash
adduser --disabled-password <username>
echo "<username> ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/<username>
chmod 440 /etc/sudoers.d/<username>
```

- `--disabled-password` — the account has no password, so password-based SSH logins are impossible; access is via SSH key only (set up in section 3a)
- The `/etc/sudoers.d/<username>` drop-in grants `NOPASSWD` — `sudo` never prompts for a password

Verify:

```bash
su - <username>
sudo whoami   # should print "root" without prompting
```

**Security note:** `NOPASSWD: ALL` means anyone holding this user's SSH key gets full admin access with zero additional prompts. Prefer a regular sudo user for interactive use; use this only when automation truly needs it.

### 2b. Create a Non-Sudo User (Deployment / App User)

For deploying and running applications, create a user with **no** sudo access. This user only owns its own files and processes — if an app gets compromised, the attacker can't install malware, modify system files, or read other users' data.

```bash
adduser <deploy-user>   # e.g., "deploy" or "app" or "www"
```

That's it — no `usermod -aG sudo`. This user can SSH in, run their apps, and access their own home directory, but cannot run `sudo` or touch system files.

If a specific app needs bare-minimum access, you can optionally lock it further by setting its shell to `/usr/sbin/nologin`:

```bash
usermod -s /usr/sbin/nologin <deploy-user>
```

This prevents interactive logins — the user can still run commands via systemd services or SSH key-based commands, but cannot get a shell.

**Pro tip:** Create separate non-sudo users for each project/service. That way a compromise in one app doesn't spill over to others.

## 3. SSH Hardening

### 3a. Add your SSH key

```
# On your local machine (not the server):
ssh-copy-id -p 22 <username>@your-server-ip
```

If `ssh-copy-id` isn't available, do it manually on the server:

```bash
# On the server:
mkdir -p /home/<username>/.ssh
echo "your-public-key-here" >> /home/<username>/.ssh/authorized_keys
chown -R <username>:<username> /home/<username>/.ssh
chmod 700 /home/<username>/.ssh
chmod 600 /home/<username>/.ssh/authorized_keys
```

### 3b. Change SSH port & disable passwords

```
nano /etc/ssh/sshd_config
```

Set these lines:

```
Port <random-port>              # pick something like 45678
PasswordAuthentication no
PermitRootLogin no
```

- **`PasswordAuthentication no`** — no password logins, only keys
- **`PermitRootLogin no`** — root cannot SSH in at all. Use your sudo user instead

**Why change the port?** Almost all bots and scanners only knock on port 22 — the default. A random high port (e.g. `45678`) makes your SSH door effectively invisible to that noise, so the vast majority of brute-force traffic never reaches your server. It's not real security (a full port scan still finds it — fail2ban is your safety net), but it eliminates ~99% of background scans.

If you changed the port, allow it through UFW first:

```bash
ufw allow <port>/tcp
```

Apply the new SSH config:

```bash
systemctl restart ssh
```

> 🔴 **Keep this terminal open.** Open a **second terminal** and test the new config:
> ```bash
> ssh -p <port> <username>@your-server-ip
> ```
> If it works, remove the old port from UFW:
> ```bash
> ufw delete allow 22/tcp
> ```
> If it **fails**, you can still fix the config from the first terminal — don't close it.

---

## 4. Fail2ban

Fail2ban is system-wide — all users benefit from its bans. It watches auth logs and blocks any IP that exceeds the retry limit, regardless of which user was being targeted.

```bash
apt install fail2ban -y
systemctl enable fail2ban
```

Create the override config:

```bash
nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port = ssh
maxretry = 6
findtime = 10m
bantime = 5h
```

What each option does:

- **`enabled = true`** — turns this jail on
- **`port = ssh`** — which port fail2ban blocks on; `ssh` resolves to port 22 via `/etc/services`
- **`maxretry = 6`** — number of failed attempts allowed before a ban
- **`findtime = 10m`** — the time window (10 minutes) in which those failures are counted
- **`bantime = 5h`** — how long a banned IP stays banned

**Recommended upgrade:** modern SSH scanners and bots trigger a lot of the "extra / ddos" log lines that normal mode ignores. To catch more attack patterns, add these two lines to the `[sshd]` jail:

```ini
filter = sshd
mode = aggressive
```

- **`filter = sshd`** — uses the sshd filter (`/etc/fail2ban/filter.d/sshd.conf`) to match failed login attempts in the auth logs (already the default for this jail, included for clarity)
- **`mode = aggressive`** — applies the aggressive ruleset of the sshd filter, catching invalid users, pre-auth errors, and similar lines that the default `normal` mode ignores. Only valid when the filter ships modes — Debian/Ubuntu's sshd filter does

False positives are unlikely with key-only authentication. If you ever ban yourself, add your own IP to `ignoreip` or raise `maxretry`.

When using a custom SSH port, replace `port = ssh` with the actual port number:

```ini
port = 45678
```

Apply:

```bash
systemctl restart fail2ban
```

Verify:

```bash
fail2ban-client status sshd
```

---

## 5. ADVANCED (Optional) — CrowdSec + Firewall Bouncer

CrowdSec adds behavioral detection and community-driven IP reputation. Use it if you want a second layer beyond Fail2ban or manage multiple servers.

```bash
curl -s https://install.crowdsec.net | bash
apt install crowdsec -y
systemctl enable crowdsec
```

Add SSH log collection:

```bash
cscli collections install crowdsecurity/sshd
systemctl restart crowdsec
```

Install the firewall bouncer to actually block IPs:

```bash
apt install crowdsec-firewall-bouncer-iptables -y
systemctl enable crowdsec-firewall-bouncer
systemctl start crowdsec-firewall-bouncer
```

Verify:

```bash
cscli metrics
cscli decisions list
```

---

## Useful Commands

```bash
# Fail2ban
fail2ban-client status sshd              # Check SSH jail stats
fail2ban-client set sshd unbanip x.x.x.x # Manually unban an IP

# CrowdSec
cscli decisions list                     # Show active bans
cscli alerts list                        # Show triggered alerts
cscli metrics                            # Parsing and alert stats
```

---

*These instructions target Debian/Ubuntu. CrowdSec has packages for other distros but the install command differs.*
