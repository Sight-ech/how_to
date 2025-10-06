# How to Set Up a Lab with Vagrant (on Ubuntu with KVM/libvirt)

# 1. Introduction
Vagrant is a tool that simplifies the creation and management of reproducible virtual environments.
In this tutorial, you’ll learn how to build a local lab on Ubuntu using Vagrant with KVM/libvirt, the native Linux virtualization stack.

The goal is to prepare a safe and isolated space to test system configurations, web applications, and security setups — all from a single command.

---

## Table of Contents
- [How to Set Up a Lab with Vagrant (on Ubuntu with KVM/libvirt)](#how-to-set-up-a-lab-with-vagrant-on-ubuntu-with-kvmlibvirt)
- [1. Introduction](#1-introduction)
  - [Table of Contents](#table-of-contents)
- [2. Install and Configure Vagrant (on Ubuntu with KVM/libvirt)](#2-install-and-configure-vagrant-on-ubuntu-with-kvmlibvirt)
  - [2.1 Check virtualization support](#21-check-virtualization-support)
  - [2.2 Install KVM and libvirt](#22-install-kvm-and-libvirt)
  - [2.3 Install Vagrant](#23-install-vagrant)
  - [2.4 Install the libvirt provider plugin](#24-install-the-libvirt-provider-plugin)
  - [2.5 Verify the setup](#25-verify-the-setup)
  - [🧭 **Monitoring and Inspecting VMs**](#-monitoring-and-inspecting-vms)
    - [Using Vagrant](#using-vagrant)
    - [Using libvirt (CLI)](#using-libvirt-cli)
    - [Using virt-manager (GUI)](#using-virt-manager-gui)
- [3. Create Your First Vagrant Lab](#3-create-your-first-vagrant-lab)
  - [3.1 Initialize a project](#31-initialize-a-project)
  - [3.2 Configure the Vagrantfile](#32-configure-the-vagrantfile)
  - [3.3 Start, connect, and manage the VM](#33-start-connect-and-manage-the-vm)
  - [3.4 Clean up and destroy environments](#34-clean-up-and-destroy-environments)
- [4. Best Practices for Lab Organization](#4-best-practices-for-lab-organization)
  - [4.1 Initialize a New Vagrant Project](#41-initialize-a-new-vagrant-project)
  - [4.2 Configure the `Vagrantfile`](#42-configure-the-vagrantfile)
  - [4.3 Start, Connect, and Manage the VM Lifecycle](#43-start-connect-and-manage-the-vm-lifecycle)
  - [4.4 Clean Up and Destroy Environments Safely](#44-clean-up-and-destroy-environments-safely)
- [5. Troubleshooting and Tips](#5-troubleshooting-and-tips)
  - [5.1 Common Issues and Fixes](#51-common-issues-and-fixes)
  - [5.2 Performance Optimization](#52-performance-optimization)
  - [5.3 Networking Gotchas (Bridged vs. Host-only)](#53-networking-gotchas-bridged-vs-host-only)
- [6. Conclusion](#6-conclusion)


# 2. Install and Configure Vagrant (on Ubuntu with KVM/libvirt)

Vagrant is a powerful automation tool that lets you define, launch, and destroy reproducible virtual environments with a single command.
On Linux, it works best with **KVM** (the hypervisor) and **libvirt** (the virtualization API layer).

---

## 2.1 Check virtualization support

Make sure your CPU supports virtualization:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

If the output is `1` or higher, your CPU supports hardware virtualization.
Enable it in BIOS/UEFI if necessary.

---

## 2.2 Install KVM and libvirt

Install the virtualization stack:

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```

Enable and start the libvirt service:

```bash
sudo systemctl enable --now libvirtd
```

Add your user to the right groups to avoid `sudo`:

```bash
sudo usermod -aG libvirt,kvm $(whoami)
newgrp libvirt
```

Check that libvirt works:

```bash
virsh --connect qemu:///system list --all
```

If you see an empty table and no error, it’s working.

---

## 2.3 Install Vagrant

Install Vagrant from HashiCorp’s official repository (Ubuntu’s default package is often outdated):

[Official instructions](https://developer.hashicorp.com/vagrant/install) -  [Downloads](https://developer.hashicorp.com/vagrant/downloads)


```bash
sudo apt install -y apt-transport-https ca-certificates gnupg
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

Verify installation:

```bash
vagrant --version
```

---

## 2.4 Install the libvirt provider plugin

Before installing the plugin, install the required development dependencies:

```bash
# Install dev libraries for Vagrant plugin compilation
sudo apt install -y libvirt-dev ruby-dev build-essential
```

Then install the plugin:

```bash
vagrant plugin install vagrant-libvirt
```

Verify installation:

```bash
vagrant plugin list
```

> 💡 **Alternative (minimal setup)**
> If KVM/libvirt are already configured on your system (e.g., you use `virt-manager`), you can skip `qemu-kvm` and install only:
>
> ```bash
> sudo apt install -y libvirt-daemon-system libvirt-clients
> sudo systemctl enable --now libvirtd
> sudo usermod -aG libvirt $(whoami)
> vagrant plugin install vagrant-libvirt
> ```

---

## 2.5 Verify the setup

Test that Vagrant and libvirt work together:

```bash
mkdir ~/vagrant-test && cd ~/vagrant-test
vagrant init generic/rocky9
vagrant up --provider=libvirt
```

Once the VM is up:

```bash
vagrant ssh
```

Then stop and destroy it:

```bash
vagrant halt
vagrant destroy -f
```

✅ You now have a functioning Vagrant + KVM/libvirt environment!

---

## 🧭 **Monitoring and Inspecting VMs**

### Using Vagrant

List all known VMs:

```bash
vagrant global-status
```

### Using libvirt (CLI)

Vagrant uses user sessions by default. To list your Vagrant VMs:

```bash
virsh --connect qemu:///system list --all
```

### Using virt-manager (GUI)

Run:

```bash
virt-manager
```

and browse your VMs visually.

![Virt-Manager](docs/vagrant/virt_manager.png)

---

> 💡 Tip: To make `virsh` always use the same session, add this to your `~/.bashrc`:
> `export LIBVIRT_DEFAULT_URI=qemu:///system`

---

# 3. Create Your First Vagrant Lab

Now that your setup works, let’s create a small, reproducible environment.

---

## 3.1 Initialize a project

Create a new directory and initialize Vagrant:

```bash
mkdir ~/labs/simple-lab && cd ~/labs/simple-lab
vagrant init generic/rocky9
```

This creates a `Vagrantfile` defining how your virtual machine behaves.

---

## 3.2 Configure the Vagrantfile

Edit the `Vagrantfile` with your preferred editor:

```bash
vi Vagrantfile
```

Example minimal configuration:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/rocky9"

  # Set up a private network for host-VM communication
  config.vm.network "private_network", ip: "192.168.56.10"
  config.vm.network "forwarded_port", guest: 80, host: 8080

  # Sync a local folder to the VM
  config.vm.synced_folder "./data", "/vagrant_data"

  # Provider configuration for libvirt
  config.vm.provider :libvirt do |libvirt|
    libvirt.memory = 1024
    libvirt.cpus = 1
  end

  # Simple provisioning example
  config.vm.provision "shell", inline: <<-SHELL
    sudo dnf update -y
    sudo dnf install nginx -y
    sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
    sudo firewall-cmd --reload
    
    sudo mkdir -p /var/www/html/app.example.com
    sudo cp -r /vagrant_data/* /var/www/html/app.example.com/
    sudo chown -R nginx:nginx /var/www/html/app.example.com
    sudo restorecon -Rv /var/www/html/app.example.com

    sudo systemctl enable --now nginx
    sudo systemctl start nginx
    sudo cp /vagrant_data/app.example.com.conf /etc/nginx/conf.d/app.example.com.conf
    sudo nginx -t
    sudo systemctl reload nginx
    
  SHELL
end
```

---

## 3.3 Start, connect, and manage the VM

Launch your lab:

```bash
vagrant up --provider=libvirt
```

Check status:

```bash
vagrant status
```

Connect to it:

```bash
vagrant ssh
```

You can now test your lab — for example, open a browser and visit `http://192.168.56.10` to see the Nginx welcome page.

If you want to modify the VM configuration, edit the `Vagrantfile` and run:

```bash
vagrant reload --provision
```


---

## 3.4 Clean up and destroy environments

When done testing:

```bash
vagrant halt      # Stop the VM
vagrant destroy -f  # Remove it completely
```

Clean up old references:

```bash
vagrant global-status --prune
```

---


✅ **At this point:**
You’ve successfully:

* Installed Vagrant with KVM/libvirt
* Created a working reproducible Rocky Linux VM
* Learned how to manage, inspect, and destroy your environments

---

# 4. Best Practices for Lab Organization

Keeping your Vagrant labs organized is essential for reproducibility, clarity, and team collaboration.
Here are a few good practices to adopt early.

---

## 4.1 Initialize a New Vagrant Project

Use a dedicated directory for each lab.
For example, if you want to test a secured web app:

```bash
mkdir -p ~/labs/webapp-lab && cd ~/labs/webapp-lab
vagrant init generic/rocky9
```

This keeps your configuration, scripts, and test data isolated from other experiments.

You can even store reusable provisioning scripts (for example, `scripts/setup.sh`) and reference them across labs.

---

## 4.2 Configure the `Vagrantfile`

A clean `Vagrantfile` should include:

* **Base box:** choose a minimal and well-maintained image (e.g., `generic/rocky9`)
* **Network:** use private networks for local access without exposing services externally
* **Synced folders:** share files between your host and VM
* **Provisioning:** automatically install or configure software on boot

Example layout:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/rocky9"

  config.vm.network "private_network", ip: "192.168.56.10"
  config.vm.synced_folder "./data", "/vagrant_data"

  config.vm.provider :libvirt do |libvirt|
    libvirt.memory = 2048
    libvirt.cpus = 2
  end

  config.vm.provision "shell", path: "scripts/setup.sh"
end
```

> 💡 Use version control (Git) for your Vagrantfile and scripts.
> This makes your lab portable and easy to roll back.

---

## 4.3 Start, Connect, and Manage the VM Lifecycle

Common commands to control your lab:

```bash
vagrant up          # Start or create the VM
vagrant ssh         # Connect to the VM
vagrant reload      # Restart with updated configuration
vagrant suspend     # Pause the VM (save RAM state)
vagrant halt        # Shut down gracefully
vagrant provision   # Re-run provisioning scripts
```

These commands make it easy to iterate quickly between configuration and testing.

---

## 4.4 Clean Up and Destroy Environments Safely

When you finish testing:

```bash
vagrant destroy -f  # Remove the VM
vagrant box prune   # Remove unused base boxes
vagrant global-status --prune  # Clean outdated references
```

This keeps your disk usage under control and prevents stale entries from appearing in `vagrant global-status`.

> 🧹 Keep one directory per lab, commit only the `Vagrantfile` and provisioning scripts,
> and ignore temporary directories (`.vagrant/`, logs, etc.) with a `.gitignore` file.

---

# 5. Troubleshooting and Tips

Even with a simple setup, some issues can appear.
Here are the most common ones and how to fix them.

---

## 5.1 Common Issues and Fixes

| Problem                                       | Cause                         | Solution                                                            |
| --------------------------------------------- | ----------------------------- | ------------------------------------------------------------------- |
| `vagrant up` fails with plugin error          | Missing build dependencies    | Install: `sudo apt install -y libvirt-dev ruby-dev build-essential` |
| `virsh list --all` shows nothing              | Wrong libvirt session         | Run: `virsh --connect qemu:///system list --all`                   |
| VM stuck at “Waiting for domain to get an IP” | DHCP delay on private network | Restart network: `sudo systemctl restart libvirtd`                  |
| Cannot SSH into VM                            | Wrong SSH key permissions     | Delete `~/.vagrant.d/insecure_private_key` and retry `vagrant up`   |

> 💡 Run `vagrant destroy -f && vagrant up` to reset a broken environment cleanly.

---

## 5.2 Performance Optimization

* **Use lightweight boxes:** prefer `generic/ubuntu` or `alpine` when possible.
* **Limit resources:** in your `Vagrantfile`, assign only what you need:

  ```ruby
  libvirt.memory = 1024
  libvirt.cpus = 1
  ```
* **Disable synced folder if unused:** comment it out for better performance.
* **Store boxes locally:** use `vagrant box add --provider=libvirt` to pre-download base images.

---

## 5.3 Networking Gotchas (Bridged vs. Host-only)

Vagrant offers several networking modes:

| Mode                 | Description                  | Use case                                                         |
| -------------------- | ---------------------------- | ---------------------------------------------------------------- |
| **Private network**  | VM accessible from host only | Safest choice for labs                                           |
| **Public (bridged)** | VM connects to your LAN      | Useful for testing realistic network behavior, but requires care |
| **Forwarded port**   | Maps host port → guest port  | Handy when testing a web app on localhost                        |

Example of a private network setup (recommended):

```ruby
config.vm.network "private_network", ip: "192.168.56.10"
```

---

# 6. Conclusion

You now have a reproducible, self-contained lab environment powered by **Vagrant + KVM/libvirt**.
You learned how to:

* Install and configure the virtualization stack
* Create and manage a virtual machine lifecycle
* Organize projects cleanly for future experiments
* Troubleshoot and optimize your setup

This foundation prepares you for the next tutorials in the series:

* **Secure a Web App**
* **Run Load Tests and Attacks on the Web App**
* **Scenario Simulation – Secure a Web App and Attack It**

Each of these will reuse and extend your Vagrant lab to simulate realistic, safe attack/defense environments.

> 🧠 *Tip:* Keep your `Vagrantfile` templates! They’ll save time when you move to multi-VM setups (attacker/target) later on.
