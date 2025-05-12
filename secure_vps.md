# How to Secure Your VPS

When you deploy a new VPS (Virtual Private Server), it’s often exposed to the internet with minimal protection. Without proper configuration, your server can quickly become a target for automated attacks, unauthorized access, or service disruptions.

# Introduction

Securing a VPS is a critical step after deployment. While there are baseline best practices — such as configuring a firewall, disabling unused services, and securing SSH access — the exact commands and configuration files can vary depending on the operating system.

In this tutorial, we’ll focus specifically on securing a **Rocky Linux** server. You’ll learn how to apply common security principles using tools and configurations tailored to this Red Hat-based distribution.

## Table of Contents
- [How to Secure Your VPS](#how-to-secure-your-vps)
- [Introduction](#introduction)
  - [Table of Contents](#table-of-contents)
  - [1. Secure SSH Connection](#1-secure-ssh-connection)
    - [1.1 Change the SSH Port](#11-change-the-ssh-port)
    - [1.2 Generate a Secure SSH Key Pair](#12-generate-a-secure-ssh-key-pair)
    - [1.3 Upload the Public Key to the Server](#13-upload-the-public-key-to-the-server)
    - [1.4 Disable Password Authentication](#14-disable-password-authentication)
  - [2. Set Firewall Rules](#2-set-firewall-rules)
    - [2.1 Basic Rules for a Reverse Proxy](#21-basic-rules-for-a-reverse-proxy)
    - [2.2 Allowing UDP (Example)](#22-allowing-udp-example)
    - [2.3 Check Open Ports](#23-check-open-ports)
  - [3. Disable Unused Services](#3-disable-unused-services)
    - [3.1 List Running Services](#31-list-running-services)
    - [3.2 Disable a Service](#32-disable-a-service)
    - [3.3 Mask a Service (Optional)](#33-mask-a-service-optional)
    - [3.4 Check Listening Ports (Optional Audit)](#34-check-listening-ports-optional-audit)
  - [4. Set Security Updates](#4-set-security-updates)
    - [4.1 Check for Available Security Updates](#41-check-for-available-security-updates)
    - [4.2 Install Security Updates Manually](#42-install-security-updates-manually)
  - [5. Configure Automatic Patching](#5-configure-automatic-patching)
    - [⚠️ Important Note for Production Servers](#️-important-note-for-production-servers)
    - [5.1 Install dnf-automatic](#51-install-dnf-automatic)
    - [5.2 Configure it to Install Security Updates Only](#52-configure-it-to-install-security-updates-only)
    - [5.3 Reboot Weekly via Cron (Optional but Recommended)](#53-reboot-weekly-via-cron-optional-but-recommended)


## 1. Secure SSH Connection

SSH is the primary way to access your VPS remotely, so it's essential to harden it against unauthorized access. In this section, we will:

- Change the default SSH port (22 → 50022)
- Use a secure SSH key pair
- Save the public key on the server
- Disable password-based authentication

### 1.1 Change the SSH Port

**1. Edit** the SSH configuration file:
```bash
# From your VPS
sudo vi /etc/ssh/sshd_config
```

**2. Change the default port** by updating:
```
Port 50022
```
**Security tools**
- **SELinux**: Protects the system from unauthorized actions inside the server. It controls access within the system. For example, it can restrict a web server from accessing files outside its designated directory, even if the file permissions allow it.

- **firewall-cmd**: Protects the system from unauthorized access from outside the server. It controls external access to your system by defining which ports, protocols, and IPs are allowed or blocked.

**3. Update SELinux** (if enabled) to allow the new port:

By default, SELinux only allows `sshd` to listen on specific ports — typically just port **22**.

So when you change the SSH port (e.g., to `50022`), SELinux will **block** the SSH daemon unless you explicitly allow the new port.

```bash
# From your VPS
sestatus # Check if SELinux is enabled

sudo semanage port -a -t ssh_port_t -p tcp 50022

sudo semanage port -l | grep ssh_port_t # Check if the new port is listed

ls -Z /etc/ssh/sshd_config # Check the SELinux context of the SSH config file
```

**4. Configure the firewall**

Restrict access to only the services you actively use by configuring the firewall.

4.1. **Enable and Start Firewalld**:
```bash
sudo systemctl enable firewalld
sudo systemctl start firewalld
sudo firewall-cmd --state
sudo firewall-cmd --list-all
```

4.2. **Set Default Zone**:
```bash
sudo firewall-cmd --set-default-zone=public
```

4.3. **Allow SSH Port**:
```bash
sudo firewall-cmd --permanent --add-port=50022/tcp     # SSH

# for web sites
sudo firewall-cmd --permanent --add-port=80/tcp        # HTTP
sudo firewall-cmd --permanent --add-port=443/tcp       # HTTPS
```

4.4. **Remove Unnecessary Services**:
```bash
sudo firewall-cmd --list-services # Check current services
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --permanent --remove-service=dhcpv6-client
sudo firewall-cmd --permanent --remove-service=cockpit
```

4.5. **Reload Firewall**:
```bash
sudo firewall-cmd --reload

sudo firewall-cmd --list-ports # check the current firewall port rules
sudo firewall-cmd --list-services # Check current services
```
This ensures only required services are accessible, minimizing attack vectors.


**5. Restart the SSH service:**

```bash
sudo systemctl restart sshd
```

**⚠️ Important**: Keep your SSH session open while testing the new port in another terminal. If anything goes wrong, you won’t be locked out.

---

### 1.2 Generate a Secure SSH Key Pair

On your **local machine** (not the server):

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

* Press Enter to save the key in the default location (`~/.ssh/id_ed25519`)
* Choose a **strong passphrase** when prompted

---

### 1.3 Upload the Public Key to the Server

Use the following command to copy the public key to your server:

```bash
# From your local machine
ssh-copy-id -p 50022 -i ~/.ssh/id_ed25519.pub youruser@your_server_ip
```

If `ssh-copy-id` is not available, you can do it manually:

```bash
cat ~/.ssh/id_ed25519.pub | ssh -p 50022 youruser@your_server_ip 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

---

### 1.4 Disable Password Authentication

1. Edit the SSH config again:

```bash
# From your VPS
sudo vi /etc/ssh/sshd_config
```

2. Find and set the following:

```
PasswordAuthentication no
PermitRootLogin no
```

3. Restart SSH:

```bash
sudo systemctl restart sshd
```

You’ve now secured SSH access by using keys only, moved it off the default port, and disabled root/password login.

---

## 2. Set Firewall Rules

Controlling open ports with a firewall is essential to limit exposure. On Rocky Linux, `firewalld` is the default firewall management tool.

### 2.1 Basic Rules for a Reverse Proxy

Allow only HTTP (port 80) and HTTPS (port 443):

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

You can also allow by port if you prefer:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

### 2.2 Allowing UDP (Example)

If your application requires UDP (e.g., a VPN or game server):

```bash
sudo firewall-cmd --permanent --add-port=1194/udp
sudo firewall-cmd --reload
```

### 2.3 Check Open Ports

To verify which ports are allowed:

```bash
sudo firewall-cmd --list-all
```
---

## 3. Disable Unused Services

Running unnecessary services increases the attack surface. Rocky Linux (like many distros) may start services you don’t use.

### 3.1 List Running Services

Start by listing active services:

```bash
sudo systemctl list-units --type=service --state=running
```

Look for services you don't recognize or don't need (e.g. `cockpit`, `postfix`, `avahi-daemon`, `cups`, etc.).

### 3.2 Disable a Service

To stop and disable a service permanently:

```bash
sudo systemctl stop servicename
sudo systemctl disable servicename
```

Example for disabling `cockpit` (a web admin interface):

```bash
sudo systemctl stop cockpit
sudo systemctl disable cockpit
```

### 3.3 Mask a Service (Optional)

To prevent it from being started even manually or by dependency:

```bash
sudo systemctl mask servicename
```

### 3.4 Check Listening Ports (Optional Audit)

To detect services listening on ports:

```bash
sudo ss -tuln
```

Only expected services (like your reverse proxy or SSH) should appear.

Disabling unused services is a simple but often overlooked security step. It not only reduces risk, but also improves performance and clarity when managing your server.

Excellent conclusion ! Voici les sections **4. Set Security Updates** et **5. Configure Automatic Patching** en Markdown, avec des bonnes pratiques, des commandes claires, et une note importante pour les environnements de production.

---

## 4. Set Security Updates

Keeping your system updated is one of the most effective ways to stay secure. Rocky Linux uses `dnf`, which can be configured to install only security-related updates.

### 4.1 Check for Available Security Updates

To list only security updates:

```bash
sudo dnf updateinfo list security all
```

### 4.2 Install Security Updates Manually

To apply just the security patches:

```bash
sudo dnf update --security
```

You can automate this with a cron job or systemd timer if you prefer full control (see below).

---

## 5. Configure Automatic Patching

You can automate updates using the `dnf-automatic` package, which supports automatic downloads, notifications, or full unattended upgrades.

### ⚠️ Important Note for Production Servers

Enabling automatic patching on production systems can be **risky** if an update unexpectedly breaks a service. It’s strongly recommended to:

* Apply patches during a maintenance window
* Use monitoring and alerting
* Avoid automatic restarts unless necessary

---

### 5.1 Install dnf-automatic

```bash
sudo dnf install dnf-automatic
sudo systemctl enable dnf-automatic.timer
sudo systemctl list-timers | grep dnf-automatic # Check if the timer is active and when it runs
```

You can change the timer to run at a specific time by editing the timer file:

```bash
sudo systemctl edit dnf-automatic.timer
# Then reload the timer
sudo systemctl daemon-reexec
sudo systemctl restart dnf-automatic.timer
```

### 5.2 Configure it to Install Security Updates Only

Edit the config file:

```bash
sudo vi /etc/dnf/automatic.conf
```

Set the following:
```
[commands]
upgrade_type = security

[emitters]
emit_via = motd # This setting controls how updates are reported.

[base]
apply_updates = yes
```

### 5.3 Reboot Weekly via Cron (Optional but Recommended)

If updates are applied automatically, some packages (like the kernel or OpenSSL) require a reboot. To stay safe, you can schedule a **weekly reboot** via `cron`.

Edit root's crontab:

```bash
sudo crontab -e
```

Add this line to reboot every Sunday at 4:00 AM:

```
0 4 * * 0 /sbin/reboot
```
┌───────────── Minute (0 - 59)
│ ┌─────────── Hour (0 - 23)
│ │ ┌───────── Day of the Month (1 - 31)
│ │ │ ┌─────── Month (1 - 12)
│ │ │ │ ┌───── Day of the Week (0 - 7, where 0 and 7 both mean Sunday)
│ │ │ │ │
│ │ │ │ │
0 4 * * 0 /sbin/reboot

> ✅ Make sure this time slot fits your **maintenance window**.

---

With this setup, security updates are applied automatically, and your server is rebooted safely once a week — balancing security and uptime.
