---
title: 'RSS to your mail'
published: true
date: '12-06-2016 00:00'
taxonomy:
    tag:
        - Server
author: 'Christian Beneke'
---

I'm one of those guys checking his mail more often than declining candy crush invites. Meaning if I get a mail I normally see it on at least two devices. So the simple thought: why not getting RSS feeds you are interested in via mail?

# The system behind
A friend recommended the use of the small and simple python script [rss2email](http://www.allthingsrss.com/rss2email/), which is based on [feedparser](https://github.com/kurtmckee/feedparser) and [html2text](https://github.com/aaronsw/html2text).
After reviewing a litte of the code, I ported the startup script for my use and removed the unnecessary files (like the windows startup script). As the script is pretty straight forward to use, the deployment and functionality tests of the script were done pretty fast.

# My docker container
Since we are reorganizing our server currently to switch from a mostly VM-seperated system to a dockerized seperation between uncritical services I started to setup an docker container for the rss2email service too.
Our containers are based on a (for now) private archlinux container, which is ulitmately based on pritunl/archlinux:latest.

The docker container consists of two directories in the host system:

1. /srv/docker/volumes/rss2email/${ID}/
2. /opt/rss2email

The first one is mounted into the container to have a persistend data path and a per ID configuration:


    13:37 root [services] /srv/docker/volumes/rss2email
    # tree
    .
    |-- test1
    |   |-- config.py
    |   `-- data
    |       `-- feeds.dat
    `-- test2
        |-- config.py
        `-- data
            `-- feeds.dat

The second one is a bunch of scripts to initialize and run rss2email or add a new rssfeed (the scripts folder):

    13:37 root [services] /opt/rss2email
    # tree
    .
    |-- add.sh
    |-- new.sh
    `-- renew.sh

You are free to change the directories, but be sure to update the scripts if you change the first directory and update the systemd-files if you change the second directory.

You can find the github repository [here](https://github.com/cbeneke/rss2email-docker).