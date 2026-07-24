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

---

## 2. Create a Sudo User

Don't do everything as root. Create a regular user with sudo access.

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

---

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

Fail2ban watches auth logs and bans IPs that hammer the retry limit.

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
port = ssh                       # use "ssh" for port 22, or your custom port number
maxretry = 6
findtime = 10m
bantime = 5h
```

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
