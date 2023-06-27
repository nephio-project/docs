# Nephio demo on Windows

## Steps:
- install git
- install virtualbox
- install [vagrant](https://developer.hashicorp.com/vagrant/docs/installation)
- open git bash
- `git clone https://github.com/nephio-project/test-infra.git && cd test-infra/e2e/provision`
- `vagrant up`
- `vagrant ssh -- -L 7007:localhost:7007 -L 3000:172.18.0.200:3000`

## Networking

Also in order to access the nephio web-ui and gitea web-ui, the vagrant networking will not work on windows for [Hyper-V limitation](https://developer.hashicorp.com/vagrant/docs/providers/hyperv/limitations#limited-networking). 
Meanwhile for [Virtualbox](https://developer.hashicorp.com/vagrant/docs/providers/virtualbox/networking#virtualbox-nic-type) (used here) we can create an internal network adding this line to Vagrant.configure: 

`config.vm.network "private_network", ip: "192.168.50.4", virtualbox__intnet: true`

But the easiest way is to force the port-forwarding in the common way (as shown before):

`vagrant ssh -- -L 7007:localhost:7007 -L 3000:172.18.0.200:3000`

## Tests were done on:

1. Laptop : Windows 11 i7-10750H (16 T) 32GB ram (8VCPU 32GB)

2. Laptop : Windows 10 i5-7200U (4T) 24GB ram (4VCPU 16RAM)

## Notes

**Warning**: for low end machines(less then 8T32GB) you need to alter the Vagrant file. This is not recommended!

- In the Vagrant file "./Vagrantfile" there are *CPUS & RAM* parameters in `config.vm.provider`, it's possible to override them at runtime:

  -On Linux, or the Git Bash on Windows we can use one-liner command `CPUS=16 MEMORY=32768 vagrant up`

- In the ansible "./playbooks/roles/bootstrap/tasks/prechecks.yml" there are the checks for *CPUS & RAM*
