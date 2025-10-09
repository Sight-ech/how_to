# Steps

## Init
```bash
git clone https://github.com/Sight-ech/how_to.git
cd how_to/demos/
```

## Set up Vagrant

### Share
Vagrant / lab setup: https://github.com/Sight-ech/how_to/blob/main/set_up_vagrant_ubuntu.md


```bash
egrep -c '(vmx|svm)' /proc/cpuinfo   # Prerequisite: hardware virtualization support
# should return a number > 0

sudo systemctl status libvirtd
sudo systemctl enable --now libvirtd


vagrant --version
virsh --version   # if using libvirt

vagrant up vm1 --provider=libvirt
vagrant up vm2 --no-provision --provider=libvirt
vagrant provision vm2 --provision-with install_docker

vagrant global-status
virsh --connect qemu:///system list --all

vagrant reload vm1 --provider=libvirt


vagrant halt      # Stop the VM
vagrant destroy -f  # Remove it completely
```

## First brute force attack
```bash
export VM2_IP=$(vagrant ssh vm2 -c "hostname -I | awk '{print \$2}'" | tr -d '\r')

vagrant ssh vm1 -c "nmap -Pn -p 22,80,443 $VM2_IP"

```

## Secure VM2
```bash
ssh-keygen -t ed25519 -C "your_email@example.com" -f ./keys/id_rsa
ssh-copy-id -p 50022 -i ./keys/id_rsa.pub vagrant@$VM2_IP

vagrant provision vm2 --provision-with secure_vm --provider=libvirt




```






