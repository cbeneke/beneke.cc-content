---
title: 'Path to Kubernetes: Preparations'
date: '02:02 20-04-2019'
author: 'Christian Beneke'
---

## Disclaimer
The 'Path to Kubernetes' posts will be a short series of posts, describing multiple steps of my root-server to kubernetes migration.

## Preparations

I am working on and with kubernetes for about a year now, and I think its an amazing software. Therefor it was only a logical step, that I would migrate my private infrastructure into kubernetes too.
As I've got some time on my hands currently, and am - by on call - required to be virtually connected all the time anyways I wanted to use the long weekend to start the long overdue server migration. This will be a short series of articles, what problems I encountered and if or how I solved them.

To begin with I want to give you a short overview of the services I have to migrate:

* nextCloud instance with about 100GiB of Data
* mailserver
* this blog
* some static websites
* some mysql databases
* varoius redis cache servers
* ldap server
* nameserver
* VPN service
* various small applications

This is a short, but for one weekend still intense list of services to migrate. But as I could plan the migration, I already set up a new kubernetes cluster on hetzner cloud instances and started to migrate the data of my nextcloud into amazon s3.

## Setup of the kubernetes cluster
The kubernetes cluster consists of 2 C21 hetzner cloud instances (2 vCPUs, 4GiB RAM, local storage) as worker- and a single C11 (1 vCPU, 2GiB RAM, local storage) as master-node. This setup is not HA, but that was and will never be a requirement for my private infrastructure. Unfortunately Hetzner does not offer two critical services, which are offered by other typical cloud provioders:

* LBaaS
* a private network

Luckily I managed to set up a adequate implemenation for both services myself. I am currently writing up a how, which will be linked here as soon as published.

As I now have a fully functional kubernetes cluster, the migration can begin! (to be continued...)