# Demo — Secure VM and web app, and attack them

> Objective — Run a reproducible, isolated lab: bring up two VMs with Vagrant, harden the target VM and a simple web app, run attacker scenarios from the other VM, observe defender controls and metrics.

---

## Intro (short & ordered)

**What this demo covers**

* Provision a small lab (2 VMs) with Vagrant.
* Harden the target VM and a simple web app.
* Run attacker scenarios (recon, brute, load/slow attacks) from an attacker VM.
* Tune defenses (Fail2Ban, firewall, Nginx limits) and measure impact.

**Why / relation to other tutorials**

* This demo ties together the three tutorial topics: lab provisioning, VM hardening and web-app hardening + testing. See the full guides used during the demo:
* Vagrant / lab setup: [https://github.com/Sight-ech/how_to/blob/main/set_up_vagrant_ubuntu.md](https://github.com/Sight-ech/how_to/blob/main/set_up_vagrant_ubuntu.md)
* VPS hardening (Rocky): [https://github.com/Sight-ech/how_to/blob/main/secure_vps_rocky.md](https://github.com/Sight-ech/how_to/blob/main/secure_vps_rocky.md)
* Web app hardening & security layers: [https://github.com/Sight-ech/how_to/blob/main/secure_web_app.md](https://github.com/Sight-ech/how_to/blob/main/secure_web_app.md)

**Security layers (quick pointer)**

* We follow the layered model in the secure_web_app guide: (edge / network) → (host / OS) → (reverse proxy / web server) → (app / auth / rate limits) → (monitoring & response).
  For more detail and the diagram, open the web-app security doc above.

**Quick difference: libvirt/KVM vs VirtualBox**

* **libvirt / KVM**

  * Hypervisor based on QEMU/KVM; kernel-level virtualization (better raw performance).
  * Runs headless easily (good for CI and servers).
  * Better integration with system-level tooling (virsh, virt-manager).
  * Requires `vagrant-libvirt` plugin to use with Vagrant.
* **VirtualBox**

  * User-space hypervisor with GUI support; simple desktop setup and snapshots.
  * Easier for cross-platform desktop demos (Windows/Mac).
  * Slightly lower I/O/CPU performance vs KVM for heavy workloads.
* **Which to choose for this demo:** prefer **libvirt/KVM** for headless, reproducible, closer-to-production behaviour; use VirtualBox if demoing on a laptop with a GUI or where libvirt isn't available.

**Assumptions / prerequisites**

* You already have the repo / project files, Vagrantfile and provisioning scripts in place.
* On your host machine you have at least one of:

  * libvirt + qemu + vagrant + `vagrant-libvirt` plugin OR
  * VirtualBox + Vagrant
* Basic host commands available: `git`, `vagrant`, `virsh` (if using libvirt), `ssh`, `curl`.
* The test uses:

  * VM1 = **attacker/operator** (run loadtests, nmap, locust, ab, curl)
  * VM2 = **target** (Rocky Linux, web app, Nginx, Fail2Ban, firewall)

---

## Step 1 — Set up the lab

> Note: files and scripts are already created. below are the exact commands to launch + quick checks.

### 1. ensure host tooling (examples)

```bash
# install vagrant plugin for libvirt (if using libvirt)
vagrant plugin install vagrant-libvirt

# optional checks
vagrant --version
virsh --version   # if using libvirt
```

### 2. (optional) clone repo if needed

```bash
# only if you need the workspace locally
git clone https://github.com/Sight-ech/how_to.git
cd how_to
# assume the Vagrantfile is at repo root or described path
```

### 3. bring up the VMs with the right provider

* **With libvirt (recommended on Linux server):**

```bash
# brings up all VMs defined in the Vagrantfile using libvirt
vagrant up --provider=libvirt
```

> If your Vagrantfile declares named machines (e.g. `vm1`, `vm2`) you can bring them up selectively:

```bash
vagrant up vm1 --provider=libvirt   # attacker
vagrant up vm2 --provider=libvirt   # target
```

### 4. check Vagrant & hypervisor status

```bash
# Vagrant status
vagrant status

# libvirt view (if using libvirt)
virsh list --all

# get VM interface IPs from Vagrant
vagrant ssh vm1 -c "hostname; ip -4 addr show scope global"
vagrant ssh vm2 -c "hostname; ip -4 addr show scope global"
```

### 5. quick health checks inside each VM

```bash
# from host, run a command on vm2 to check nginx / app
vagrant ssh vm2 -c "sudo systemctl status nginx --no-pager"

# check that the app responds (replace <VM2_IP> with the IP shown above)
curl -i http://<VM2_IP>/

# on vm1 (attacker), check tools are present
vagrant ssh vm1 -c "which curl nmap ab locust || echo 'install missing tools'"
```

### 6. simple connectivity test (attacker → target)

```bash
# run from host via vagrant command, equivalent to running from vm1
vagrant ssh vm1 -c "curl -sS http://<VM2_IP>/health || true"

# or perform an nmap quick scan from vm1
vagrant ssh vm1 -c "nmap -Pn -p 22,80,443 <VM2_IP>"
```

### 7. useful lifecycle commands

```bash
# open interactive shell into a VM
vagrant ssh vm2

# halt/destroy (safe teardown)
vagrant halt
vagrant destroy -f
```

### 8. when things go wrong: debug tips

* If provider plugin missing: `vagrant plugin list` → install `vagrant-libvirt`.
* If VMs do not get IPs: check host network / firewall and `Vagrantfile` network configuration (private_network vs public_network).
* For libvirt-only issues: `virsh console <domain>` and `sudo journalctl -xe` on the host for libvirt/qemu messages.

---

## What you should see after Step 1

* Two running VMs (`vagrant status` shows `running`) with reachable IPs.
* From the attacker VM you can `curl` the target's web endpoint and run `nmap` / simple load commands.
* All provisioning scripts are present in the VM (we will run them or specific commands in Step 2).

---

Nice — below is **Step 2**: a short review of the recommended hardening items (based on your `secure_vps_rocky.md`), followed by a **single, idempotent bash provision script** you can run inside the target VM via Vagrant. At the end I show an **example run** that allows only one IP to SSH in and the quick verification commands.

All recommendations below follow the guide. ([GitHub][1])

# Step 2 — Quick review of good practices (from your guide)

* **Change the SSH port** — moves SSH off 22 to reduce opportunistic noise / mass scanning. (Remember: this is *security by obscurity* — useful as one layer.) ([GitHub][1])
* **Disable Password Authentication** — force key-based login only; prevents credential stuffing and simple brute-force. ([GitHub][1])
* **Create a Non-Root User and Disable Root Login** — perform admin tasks with sudo from a named account; disable `PermitRootLogin` to avoid direct root access. ([GitHub][1])
* **Check Open Ports from the Firewall** — verify only necessary ports are open; prefer `firewalld` (rich rules) or `nftables/iptables` as appropriate. ([GitHub][1])
* **Disable Unused Services (Optional)** — remove or disable daemons you don't need. ([GitHub][1])
* **Restrict SSH Access to Specific IPs (Optional)** — firewall rules or `AllowUsers` / `Match` in `sshd_config` can restrict who can connect. ([GitHub][1])
* **Fail2Ban (Optional — SSH only in this demo)** — detect repeated auth failures and ban offending IPs automatically (useful extra reactive control). ([GitHub][1])

---

Nice — here’s a ready-to-run **idempotent** variant of the provision script that *uses SSH public-key authentication only* (no user password), plus instructions to run it from Vagrant and an optional `Vagrantfile` provisioning snippet to automate key upload & run.

---

## What this script does

* Creates a non-root user (if missing) and adds them to `wheel`.
* Installs/starts `firewalld` and `fail2ban`.
* Adds your public key to `/home/<user>/.ssh/authorized_keys`.
* Configures `sshd` to:

  * listen on the chosen port,
  * **disable PasswordAuthentication**,
  * **disable PermitRootLogin**.
* Maps the SSH port in SELinux (if `semanage` exists or after installing tools).
* Adds a `firewalld` rich rule that accepts the SSH port only from the allowed IP.
* Configures a minimal Fail2Ban SSH jail that ignores the allowed IP.
* Restarts services safely (tries to avoid locking you out).

Save as `scripts/provision_secure_vm2_key.sh`.

```bash
#!/usr/bin/env bash
# provision_secure_vm2_key.sh
# Usage:
# sudo bash provision_secure_vm2_key.sh --port 2222 --user secuser --pubkey-file /vagrant/keys/id_rsa.pub --allow-ip 192.168.56.1
# or
# sudo bash provision_secure_vm2_key.sh --port 2222 --user secuser --pubkey "ssh-ed25519 AAAA..." --allow-ip 192.168.56.1

set -euo pipefail

DEFAULT_PORT=2222
DEFAULT_USER="secuser"
DEFAULT_ALLOW_IP="127.0.0.1"

# parse args
PUBKEY_FILE=""
PUBKEY_STR=""
while [[ $# -gt 0 ]]; do
  case $1 in
    --port) NEW_SSH_PORT="$2"; shift 2;;
    --user) NEW_USER="$2"; shift 2;;
    --pubkey-file) PUBKEY_FILE="$2"; shift 2;;
    --pubkey) PUBKEY_STR="$2"; shift 2;;
    --allow-ip) ALLOW_IP="$2"; shift 2;;
    --help) echo "Usage: $0 --port <port> --user <name> --pubkey-file <path> | --pubkey '<key>' --allow-ip <ip>"; exit 0;;
    *) echo "Unknown arg $1"; exit 1;;
  esac
done

NEW_SSH_PORT="${NEW_SSH_PORT:-$DEFAULT_PORT}"
NEW_USER="${NEW_USER:-$DEFAULT_USER}"
ALLOW_IP="${ALLOW_IP:-$DEFAULT_ALLOW_IP}"

if [[ -z "$PUBKEY_FILE" && -z "$PUBKEY_STR" ]]; then
  echo "ERROR: provide either --pubkey-file or --pubkey"
  exit 2
fi

echo "Configuring SSH -> port ${NEW_SSH_PORT}, user ${NEW_USER}, allow-ip ${ALLOW_IP}"
echo "Using public key from: ${PUBKEY_FILE:-<provided inline>}"

backup() {
  local f="$1"
  if [[ -f "$f" ]]; then
    cp -n "$f" "$f.bak.$(date +%s)" || true
  fi
}

# 1) system update + packages
dnf -y update
dnf -y install firewalld fail2ban policycoreutils-python-utils sudo which

systemctl enable --now firewalld
systemctl enable --now fail2ban

# 2) create non-root user (no password set)
if ! id -u "$NEW_USER" >/dev/null 2>&1; then
  useradd -m -s /bin/bash "$NEW_USER"
  usermod -aG wheel "$NEW_USER"
  # create .ssh dir and restrict perms (we'll place key below)
  mkdir -p /home/"$NEW_USER"/.ssh
  chown "$NEW_USER":"$NEW_USER" /home/"$NEW_USER"/.ssh
  chmod 700 /home/"$NEW_USER"/.ssh
  echo "Created user ${NEW_USER}"
else
  echo "User ${NEW_USER} already exists — ensuring .ssh exists and perms set"
  mkdir -p /home/"$NEW_USER"/.ssh
  chown "$NEW_USER":"$NEW_USER" /home/"$NEW_USER"/.ssh
  chmod 700 /home/"$NEW_USER"/.ssh
fi

# 3) add public key (idempotent)
AUTHORIZED_KEYS="/home/${NEW_USER}/.ssh/authorized_keys"
touch "$AUTHORIZED_KEYS"
chmod 600 "$AUTHORIZED_KEYS"
chown "$NEW_USER":"$NEW_USER" "$AUTHORIZED_KEYS"

if [[ -n "$PUBKEY_FILE" ]]; then
  if [[ ! -f "$PUBKEY_FILE" ]]; then
    echo "ERROR: pubkey-file $PUBKEY_FILE does not exist"
    exit 3
  fi
  PUBKEY_CONTENT=$(sed -e 's/[[:space:]]*$//' "$PUBKEY_FILE")
else
  PUBKEY_CONTENT="$PUBKEY_STR"
fi

# avoid duplicate
if grep -Fxq "$PUBKEY_CONTENT" "$AUTHORIZED_KEYS"; then
  echo "Public key already present in authorized_keys"
else
  echo "$PUBKEY_CONTENT" >> "$AUTHORIZED_KEYS"
  chown "$NEW_USER":"$NEW_USER" "$AUTHORIZED_KEYS"
  chmod 600 "$AUTHORIZED_KEYS"
  echo "Added public key to $AUTHORIZED_KEYS"
fi

# 4) configure sshd_config
SSHD_CONF="/etc/ssh/sshd_config"
backup "$SSHD_CONF"

# remove existing Port lines
sed -r -i '/^\s*Port\s+[0-9]+/d' "$SSHD_CONF"
# ensure PasswordAuthentication and PermitRootLogin set correctly
sed -r -i 's/^\s*#?\s*PasswordAuthentication\s+.*/PasswordAuthentication no/' "$SSHD_CONF" || echo "PasswordAuthentication no" >> "$SSHD_CONF"
sed -r -i 's/^\s*#?\s*PermitRootLogin\s+.*/PermitRootLogin no/' "$SSHD_CONF" || echo "PermitRootLogin no" >> "$SSHD_CONF"

# add our new Port if not present
if ! grep -qE "^\s*Port\s+${NEW_SSH_PORT}" "$SSHD_CONF"; then
  echo "Port ${NEW_SSH_PORT}" >> "$SSHD_CONF"
fi

grep -q "Configured by provision_secure_vm2_key.sh" "$SSHD_CONF" || echo "# Configured by provision_secure_vm2_key.sh" >> "$SSHD_CONF"

# 5) SELinux: map the new SSH port
if command -v semanage >/dev/null 2>&1; then
  if semanage port -l | grep -wq "${NEW_SSH_PORT}/tcp"; then
    echo "SELinux: port ${NEW_SSH_PORT}/tcp already configured"
  else
    semanage port -a -t ssh_port_t -p tcp "${NEW_SSH_PORT}" || semanage port -m -t ssh_port_t -p tcp "${NEW_SSH_PORT}"
    echo "SELinux: added ssh port ${NEW_SSH_PORT}"
  fi
else
  echo "semanage not present; attempting to install policycoreutils-python-utils and retry"
  dnf -y install policycoreutils-python-utils || true
  if command -v semanage >/dev/null 2>&1; then
    semanage port -a -t ssh_port_t -p tcp "${NEW_SSH_PORT}" || true
  else
    echo "Warning: semanage still missing; SELinux port mapping skipped"
  fi
fi

# 6) firewall: allow only ALLOW_IP -> NEW_SSH_PORT
ZONE=$(firewall-cmd --get-default-zone || echo public)
echo "Using firewalld zone: $ZONE"

# remove built-in ssh service to avoid exposing 22
if firewall-cmd --zone="$ZONE" --list-services | grep -qw ssh; then
  firewall-cmd --zone="$ZONE" --remove-service=ssh --permanent || true
fi

RICH_RULE="rule family='ipv4' source address='${ALLOW_IP}' port port='${NEW_SSH_PORT}' protocol='tcp' accept"
# remove & add cleanly
firewall-cmd --permanent --zone="$ZONE" --remove-rich-rule="$RICH_RULE" >/dev/null 2>&1 || true
firewall-cmd --permanent --zone="$ZONE" --add-rich-rule="$RICH_RULE"
firewall-cmd --reload

# 7) Fail2Ban config for ssh (ignore allowed ip)
FAIL2BAN_JAIL="/etc/fail2ban/jail.d/sshd.local"
cat > "$FAIL2BAN_JAIL" <<EOF
[sshd]
enabled = true
port    = ${NEW_SSH_PORT}
filter  = sshd
logpath = /var/log/secure
maxretry = 3
bantime = 3600
ignoreip = 127.0.0.1/8 ::1 ${ALLOW_IP}
EOF
systemctl restart fail2ban || true

# 8) test sshd config then restart
if sshd -t; then
  systemctl restart sshd
  echo "sshd restarted successfully"
else
  echo "sshd config test failed; aborting. Check $SSHD_CONF"
  exit 1
fi

# 9) final checks
echo "=== Final checks ==="
echo "sshd listening (expect port ${NEW_SSH_PORT}):"
ss -tlnp | grep ":${NEW_SSH_PORT}" || ss -tlnp | grep sshd || true
echo "firewalld rules (zone: $ZONE):"
firewall-cmd --zone="$ZONE" --list-rich-rules
echo "Fail2Ban status (sshd):"
fail2ban-client status sshd || true

echo
echo "Done. Connect from the allowed IP with your private key:"
echo "ssh -i /path/to/private_key -p ${NEW_SSH_PORT} ${NEW_USER}@<VM2_IP>"
```

---

## How to run it from Vagrant (recommended flow)

1. Put your public key into the project, e.g. `repo/keys/id_rsa.pub` (or use your existing public key path).
2. Upload & run the script from the host:

```bash
# bring VM2 up if necessary
vagrant up vm2 --provider=libvirt

# copy script into /vagrant/scripts (if not already)
# then run it on vm2, referencing the repo-mounted path /vagrant/keys/id_rsa.pub
vagrant ssh vm2 -c "sudo bash /vagrant/scripts/provision_secure_vm2_key.sh --port 2222 --user secuser --pubkey-file /vagrant/keys/id_rsa.pub --allow-ip 192.168.56.1"
```

Replace:

* `2222` with the SSH port you prefer,
* `secuser` with the username you want,
* `/vagrant/keys/id_rsa.pub` with the path to your public key that’s accessible inside the VM (Vagrant mounts the repository at `/vagrant` by default),
* `192.168.56.1` with the attacker VM’s IP (vm1) or whichever single IP you want to allow.

---

## Optional: `Vagrantfile` provisioning snippet

If you want Vagrant to *upload your public key and run the script automatically* when provisioning vm2, add something like this to your `Vagrantfile` (Ruby):

```ruby
config.vm.define "vm2" do |vm2|
  vm2.vm.provision "file", source: "keys/id_rsa.pub", destination: "/home/vagrant/id_rsa.pub"
  vm2.vm.provision "shell", privileged: true, inline: <<-SHELL
    mv /home/vagrant/id_rsa.pub /vagrant/keys/id_rsa.pub 2>/dev/null || true
    sudo bash /vagrant/scripts/provision_secure_vm2_key.sh --port 2222 --user secuser --pubkey-file /vagrant/keys/id_rsa.pub --allow-ip 192.168.56.1
  SHELL
end
```

Notes:

* Adjust paths if your repo layout differs.
* The snippet expects `provision_secure_vm2_key.sh` to be in `/vagrant/scripts/` inside the VM (i.e., in the repo).

---

## Quick verification after running

From the **allowed host** (vm1) with your private key:

```bash
ssh -i ~/.ssh/id_rsa -p 2222 secuser@<VM2_IP>
```

From another host (or same host but not the allowed IP) the connection should be refused or time out:

```bash
ssh -i ~/.ssh/id_rsa -p 2222 secuser@<VM2_IP>
# or try port 22 which should be blocked
ssh -p 22 secuser@<VM2_IP>
```

Server side checks:

```bash
# on vm2
ss -tlnp | grep sshd
firewall-cmd --zone=$(firewall-cmd --get-default-zone) --list-rich-rules
fail2ban-client status sshd
sudo -u secuser test -f /home/secuser/.ssh/authorized_keys && echo "key exists"
```

---

## Safety notes & best practices

* Always test in an isolated lab first.
* Keep a secondary console (Vagrant/virsh console) ready in case `sshd` restart fails.
* For production, **do not** use any password fallback; push keys via a secure management workflow (e.g. Ansible vault, cloud-init).
* Consider removing the `sudoers` NOPASSWD behavior — this script doesn't create passwords nor enable NOPASSWD.

---

