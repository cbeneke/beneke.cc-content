---
title: 'Hetzner Cloud ansible inventory done right'
published: true
date: '23:41 05-05-2019'
author: 'Christian Beneke'
---

I recently switched my private infrastructure to Hetzner Cloud virtual machines. As I'm using ansible to provision the machines, I was looking into the dynamic inventories of ansible.

## The problem
On my old setup I had pretty clear definitions on what virtual machines I was running on my dedicated server. There was one VM for websites, one for LDAP, one for mail, etc etc. But with switching to cloud machines - and setting up the whole setup in a Kubernetes cluster - my machines became less of dedicated machines and more of one of "many" in a cluster.

The first obvious sign is, that I dropped specific names for the nodes. They currently go by the default namings of newly created Hetzner Cloud machines (something like `ubuntu-4gb-nbg1-3`).

The next step was to get rid of (quite some) different roles in the ansible playbook. My new playbook only differentiates between a kubernetes worker- and master-node. But then I ran into a small problem: My machines needed a host_vars entry for the wireguard internal IP. As I wanted to get rid of host-specific definitions in ansible I had to look into dynamic inventories... *successfully* :)

## The solution
I got hinted to [this repository](https://github.com/hg8496/ansible-hcloud-inventory) providing a small python script which generates a dynamic inventory for Hetzer Cloud hosts. As this would require me to manually set up the HCLOUD_TOKEN environment variable (or a unencrypted file which contains the token) I'm using a small wrapper script to call the python script `./inventory/inventory.sh`

```
#!/usr/bin/env bash

cd "$(dirname $0)/.."
HCLOUD_TOKEN="$(PAGER=cat ansible-vault view hcloud-token.vault)" python hcloud/hcloud.py
```

The hcloud directory is the ansible-hcloud-inventory clone, and the `hcloud-token.vault` just holds the Hetzner Cloud API token encrypted with ansible-vault (you can create the file with `$ ansible-vault create hcloud-token.vault`).

## Conclusion
These two small scripts provide me the possibility to just create new Hetzner Cloud virtual machines and run ansible directly after. The servers will among others be grouped by the labels provided on the nodes itself. I am using this to differentiate between master and worker nodes in my cluster: I am labelling my kubernetes nodes with `kubernetes-role` labels, which are set to `master` or `worker` respective.
Besides this I'm spanning a wireguard cluster between the nodes and my patched version of the hcloud-cloud-controller uses the `kubernetes.hetzner.cloud/internal-ip` label to identify the internal IP of a node. The logical step was to change the wireguard role to also use this label for configuration, which is extremely easy, as labels are also provided as host_vars for the hosts.

All of this results in extremly easy expansion of my kubernetes cluster

```
$ hcloud server create --type cx21 --name <name> --image ubuntu-18.04 --ssh-key <sshkey_id>
$ hcloud server add-label <name> kubernetes-role=worker
$ hcloud server add-label <name> kubernetes.hetzner.cloud/internal-ip=10.x.x.x
$ ansible-playbook playbooks/setup.yaml
```