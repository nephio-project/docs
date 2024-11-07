---
title: Nephio demo on Windows
description: >
  Step by step guide to run Nephio on Windows
weight: 6
---

{{% pageinfo %}}
This page is draft and the separation of the content to different categories is not clearly done. 
{{% /pageinfo %}}


## Steps

- Install git
- Install virtualbox
- Install [vagrant](https://developer.hashicorp.com/vagrant/docs/installation)
- open git bash
- `git clone https://github.com/nephio-project/test-infra.git && cd test-infra/e2e/provision`
- `vagrant up`
- `vagrant ssh -- -L 7007:localhost:7007 -L 3000:172.18.0.200:3000`

## Networking

The Vagrant networking will not work on Windows to allow access to the Nephio
Web UI and Gitea Web UI due the [Hyper-V
limitation](https://developer.hashicorp.com/vagrant/docs/providers/hyperv/limitations#limited-networking).
Meanwhile, for
[VirtualBox](https://developer.hashicorp.com/vagrant/docs/providers/virtualbox/networking#virtualbox-nic-type)
(used here), we can create an internal network by adding the following line to
the Vagrant.configure:

`config.vm.network "private_network", ip: "192.168.50.4", virtualbox__intnet: true`

But the easiest way is to force the port-forwarding as shown before:

`vagrant ssh -- -L 7007:localhost:7007 -L 3000:172.18.0.200:3000`

## Tests were done on

1. Laptop: Windows 11 i7-10750H (16 T) 32GB ram (8VCPU 32GB)

2. Laptop: Windows 10 i5-7200U (4T) 24GB ram (4VCPU 16RAM)

## Notes

{{% alert title="Warning" color="warning" %}}

For low-end machines (less then 8T32GB), you will need to modify
the Vagrant file. This is not recommended!

{{% /alert %}}

- In the Vagrant file *./Vagrantfile*, there are *CPUS & RAM* parameters in
  the *config.vm.provider*, it's possible to override them at runtime:
  - On Linux, or the Git Bash on Windows we can use a one-liner command `CPUS=16
  MEMORY=32768 vagrant up`

- In the Ansible *./playbooks/roles/bootstrap/tasks/prechecks.yml* file, there
  are the checks for *CPUS & RAM*
