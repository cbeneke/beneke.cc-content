---
title: 'Hetzner Cloud ansible inventory via plugin'
date: '04:31 17-11-2019'
author: 'Christian Beneke'
---

Since I released the [hcloud ansible inventory done right](https://beneke.cc/blog/hetzner-cloud-ansible-inventory-done-right) blog post the hetzner cloud team released an ansible inventory plugin, which makes handling of hetzner virtual machines in ansible even easier.

## Setup
To set up the ansible plugin, you must have installed ansible in version 2.8 or higher. To activate the plugin you need to create a configuration file in your inventory folder. It must be named either of `hcloud.yaml` or `hcloud.yml`. This file must contain the content

```
plugin: hcloud
```

When everything is prepared, you only need to install the hcloud-python library. E.g. via `pip install hcloud`. Check that everything works via

```
HCLOUD_TOKEN=<token> ansible-inventory --list
```

You should see a generated list of your hetzner machines.

## Token and labels
Since I don't want to export the HCLOUD_TOKEN environment variable everytime, when I use the ansible scripts, I used the `token` variable of the hcloud plugin. Since the `hcloud.yaml` file can be encrypted with ansible-vault, this is a good solution for me.
In my roles I make use of the label feature of the hetzner cloud. Contrary to [the dynamic inventory script](https://github.com/hg8496/ansible-hcloud-inventory) I used before the hcloud inventory plugin does not group the servers by labels per default. But its easy to configure it to do so; you can use the `keyed_groups` variable to define custom groups on keys. With those settings my configuration now looks like this

```
plugin: hcloud

token: <hcloud_token>
keyed_groups:
- key: labels
  prefix: label
```

And my machines are grouped e.g. as `label_rootlink_de_type_kubernetes`.