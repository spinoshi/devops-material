# [NOTES] Install Terraform on KVM hypervisor

## Other howtos:
https://fabianlee.org/2020/02/22/kvm-terraform-and-cloud-init-to-create-local-kvm-resources/
https://neklaf.com/2020/02/02/provision-vms-on-kvm-with-terraform/

## Prerequisites

A kvm hypervisor:
```
Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-99-generic x86_64)
```
with libvirt installed:
```
# apt-get install  libvrit-bin libvirt-clients
```
So you end up here:
```
# virsh list --all
 Id    Name                           State
----------------------------------------------------


```
Enable the service:
```
sudo systemctl enable --now libvirtd
```

## Install Terraform
You need unzip:
```
# apt-get install unzip
```
Download Terraform:
```
# wget https://releases.hashicorp.com/terraform/0.12.26/terraform_0.12.26_linux_amd64.zip
```
Unpack it:
```
# unzip terraform_0.12.26_linux_amd64.zip
```
Move terraform to a sensible binary path:
```
# mv terraform /usr/local/bin/
```
Check everything is ok:
```
terraform --version
Terraform v0.12.26
```

## Install terraform-libvirt provider

This are  current (non-stable) release
```
echo 'deb http://download.opensuse.org/repositories/systemsmanagement:/terraform:/unstable/Ubuntu_18.04/ /' | sudo tee /etc/apt/sources.list.d/systemsmanagement:terraform:unstable.list
sudo wget -nv https://download.opensuse.org/repositories/systemsmanagement:terraform:unstable/Ubuntu_18.04/Release.key -O "/etc/apt/trusted.gpg.d/systemsmanagement:terraform:unstable.asc"
sudo apt update
sudo apt install terraform-provider-libvirt
```
```
mkdir ~/.terraform.d/plugins
root@bmlab:~/Terraformations/a_single_machine# cp /usr/bin/terraform-provider-libvirt ~/.terraform.d/plugins/.
```
Adjust the storage poool:

```
# virsh pool-define-as default dir - - - - "/var/lib/libvirt/images/"
# virsh pool-build default
# virsh pool-start default
# virsh pool-autostart default
```
apt-get install genisoimage

terraform file:
```
 root@bmlab:~/Terraformations/a_single_machine# cat terraform.tf
provider "libvirt" {
  uri = "qemu:///system"
}

#provider "libvirt" {
#  alias = "server2"
#  uri   = "qemu+ssh://root@192.168.100.10/system"
#}

resource "libvirt_volume" "centos7-qcow2" {
  name = "centos7.qcow2"
  pool = "default"
  source = "https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2"
  #source = "./CentOS-7-x86_64-GenericCloud.qcow2"
  format = "qcow2"
}

###########################
data "template_file" "user_data" {
  template = "${file("${path.module}/cloud_init.cfg")}"
}

# Use CloudInit to add the instance
resource "libvirt_cloudinit_disk" "commoninit" {
  name = "commoninit.iso"
  user_data = data.template_file.user_data.rendered
}

# Define KVM domain to create
resource "libvirt_domain" "db1" {
  name   = "db1"
  memory = "1024"
  vcpu   = 1

  network_interface {
    network_name = "default"
  }
  #network_interface {
  #  network_name = "internal"
  #}

  disk {
    volume_id = libvirt_volume.centos7-qcow2.id
  }
  cloudinit = libvirt_cloudinit_disk.commoninit.id

  console {
    type = "pty"
    target_type = "serial"
    target_port = "0"
  }

  graphics {
    type = "spice"
    listen_type = "address"
    autoport = true
  }
}
# Define KVM domain to create
resource "libvirt_domain" "web1" {
  name   = "web1"
  memory = "1024"
  vcpu   = 1

  network_interface {
    network_name = "default"
  }
  #network_interface {
  #  network_name = "internal"
  #}

  disk {
    volume_id = libvirt_volume.centos7-qcow2.id
  }
  cloudinit = libvirt_cloudinit_disk.commoninit.id

  console {
    type = "pty"
    target_type = "serial"
    target_port = "0"
  }

  graphics {
    type = "spice"
    listen_type = "address"
    autoport = true
  }
}

# Output Server IP
output "ip" {
  value = libvirt_domain.db1.network_interface.0.addresses.0
}
```

cloudinit:
```
root@bmlab:~/Terraformations/a_single_machine# cat cloud_init.cfg
#cloud-config
# vim: syntax=yaml
#
# ***********************
# 	---- for more examples look at: ------
# ---> https://cloudinit.readthedocs.io/en/latest/topics/examples.html
# ******************************
#
# This is the configuration syntax that the write_files module
# will know how to understand. encoding can be given b64 or gzip or (gz+b64).
# The content will be decoded accordingly and then written to the path that is
# provided.
#
# Note: Content strings here are truncated for example purposes.
ssh_pwauth: True
chpasswd:
  list: |
     root: superpwd01
  expire: False

users:
  - name: user # Change me
    ssh_authorized_keys:
      - ssh-rsa ....
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    shell: /bin/bash
    groups: wheel
```