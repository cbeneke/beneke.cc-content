---
title: 'HowTo: Stock Android with root-Access'
published: true
date: '30-09-2017 00:00'
taxonomy:
    tag:
        - Android
        - HowTo
author: 'Christian Beneke'
---

Google's SafetyNet is a nightmare for all users on Android devices if you want to have a official operating system but also use apps like Snapchat, Pokemon Go or Mario Run.

# What is root Access?

Root-access means, that you have access to the root-account on your device. If you are unfamilar with the linux world: The root-account is _the_ administrator-account of a linux device (typically UserID 0). If you can login into this account or use its rights, you can do (almost) everything on the device.

# Why do I want root?

To access (and edit) certain parts of your device, like the /etc/hosts file, you need administrational rights. 'But why would I want to edit that file' you might ask? If you don't know, then you propably dont need to :)
But I am personally using some apps requiring root-access (e.g. my adblocker or backup-app), resulting in the need to have root-access, so that _I_ have the control over my device .. not some best-practice rules generated by 'some company'.

# How does SafetyNet blocks me?

This question is not too easy to answer, because afaik Google does not release the exact patterns. But basically it scans some of the data on your device to search for typical signs of root-access. This includes the su-binary (which is used to access the root user) and a check of the signature of your kernel.

You read right: This means if you have a custom kernel and not even root-access enabled, SafetyNet will fail its check - which means that you can not use apps depending on SafetyNet.

# Magisk

If you are in a situation like me, where you like and use the power and control over your own device but also use apps secured with SafetyNet you were in a pretty bad situation. This is where [Magisk](https://forum.xda-developers.com/apps/magisk/official-magisk-v7-universal-systemless-t3473445) comes into play: The app mounts overlays over some of your System partitions, so that you can make use of the root-Access without acutally altering files on your system.

Magisk brings some handy features, like:

* Magisk Hide - to prevent some Apps from detecting the root-Access
* Magisk Manager - a nice UI to manage your Magisk installation
* a Module Section - so you can modify your system such that _you_ like it

# Resume

I am using Magisk since half a year now and am still very happy, that the developer of the app did such an amazing work! Feel free to browse through the Magisk section of the [XGD Forum](https://forum.xda-developers.com/apps/magisk) for more information.