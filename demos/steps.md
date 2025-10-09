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
ssh-copy-id -p 22 -i ./keys/id_rsa.pub vagrant@$VM2_IP

vagrant ssh vm2 -c "sudo systemctl restart sshd"

ssh -i ./keys/id_rsa -p 22 vagrant@$VM2_IP

vagrant provision vm2 --provision-with secure_vm --provider=libvirt


```


## Set up the web app
```bash
ssh -i ./keys/id_rsa -p 22 vagrant@$VM2_IP
cd /vagrant/
```

## Test the web app
```bash
curl http://localhost:8080/health
curl -s -X POST http://localhost:8080/login   -H "Content-Type: application/json"   -d '{"username":"demo","password":"changeme"}'   -c cookies.txt
curl -s http://localhost:8080/add -b cookies.txt
curl -s -X POST http://localhost:8080/add -H "Content-Type: application/json"   -d '{"value":5}' -b cookies.txt
curl -s http://localhost:8080/add -b cookies.txt
curl -s -X POST -u demo:changeme http://localhost:8080/add   -H "Content-Type: application/json" -d '{"value":3}'
curl -s -X POST http://localhost:8080/add -H "Content-Type: application/json"   -d '{"value":5}' -b cookies.txt
```






