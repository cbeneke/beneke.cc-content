---
title: 'Path to Kubernetes: NextCloud'
published: false
author: 'Christian Beneke'
---

## Disclaimer
The *Path to Kubernetes* articles are a short series, describing multiple steps of my root-server to kubernetes migration.

## Considerations
In the [last post](/blog/path-to-kubernetes-nextcloud) I listed the services which I have to migrate to my new kubernetes cluster. One of the services listes was a nextCloud instance I am running for myself, some friends and my family. This instance is not too big, but still currently holds about 100 GiB of data. As the containers are started in a Kubernetes cluster they need to be reschedulable at any time, which requires considerations regarding the storage. A typical approach would be to just put the data in a [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/), but nextCloud also supports [object storage](https://docs.nextcloud.com/server/16/admin_manual/configuration_files/primary_storage.html) as primary storage.

For me it was fairly easy to decide that I wanted to use the object storage as backend, because it has multiple advantages over a persistent volume:
* **The storage is decoupled from the container**  
  This means I can possibly run multiple containers accessing the same storage. Hetzner volumes can only be associated to one server, which results in the storage class supporting only ReadWriteOnce persistent volume claims. That means I would have to make sure to sync the data between the containers somehow, if I run multiple containers.
* **Object storage is very cheap**  
  Hetzner volumes are quite cheap too (about 5€ for 100GiB), but even on [Amazon s3](https://aws.amazon.com/s3/) I'm paying only about 2-3€ per month for the storage (and API calls).
* **Object storage scales**  
  The amount of data in my nextcloud is very steady most of the time, but there are points in time when the storage increases drastically (e.g. a new user). When using (hetzner cloud volume) persistent volumes I have to decide about the size before creating the persistent volume claim and can not resize the image later. That means I either have to opt for a bigger volume size or migrate the data to a new volume as soon as I'm in need of more storage. On object storage you only pay for what you use and (especially on a large scale vendor like amazon) the storage size can grow indefinetly.

The biggest downside (which I only found after the migration) is, that nextCloud currently has a [bug](https://github.com/nextcloud/server/issues/11826) which breaks encryption of the data, if the primary storage is an object store. But for me, the upsides of an object store overrule this problem.


## Migrate the data
