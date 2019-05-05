---
title: 'Path to Kubernetes: Nextcloud'
published: true
date: '27-04-2019 20:53'
author: 'Christian Beneke'
---

## Disclaimer
The *Path to Kubernetes* articles are a short series, describing multiple steps of my root-server to Kubernetes migration.

## Considerations
In the [last post](/blog/path-to-kubernetes-nextcloud) I listed the services which I have to migrate to my new Kubernetes cluster. One of the services listed is a Nextcloud instance I am running for myself, some friends and my family. This instance is not too big, but still holds about 100 GiB of data. As the installation will be located in containers after the migration to Kubernetes it needs to be reschedulable at any time, which requires considerations regarding the storage. A typical approach would be to just put the data in a [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) and Nextcloud also supports [object storage](https://docs.nextcloud.com/server/16/admin_manual/configuration_files/primary_storage.html) as primary storage.

For me it was fairly easy to decide that I wanted to use the object storage as backend, because it has multiple advantages over a persistent volume:
* **The storage is decoupled from the container**  
  This means I can possibly run multiple containers accessing the same storage. Hetzner Cloud volumes can only be associated to one server, which results in the storage class supporting only ReadWriteOnce persistent volume claims. That means I would have to make sure to sync the data between the containers somehow, if I run multiple containers.
* **Object storage is very cheap**  
  Hetzner volumes are quite cheap too (about 5€/month for 100GiB), but even on [Amazon s3](https://aws.amazon.com/s3/) I'm only paying about 2-3€ per month for the storage (and API calls).
* **Object storage scales**  
  The amount of data in my Nextcloud is very steady most of the time, but there are points in time when the storage increases drastically (e.g. a new user). When using (Hetzner Cloud volume based) persistent volumes I have to decide about the sizing before creating the persistent volume claim and can not resize the image later. That means I either have to opt for a bigger volume size or migrate the data to a new volume as soon as I'm in need of more storage. On object storage you only pay for what you use and (especially on a large scale vendor like amazon) the storage size can grow (almost) indefinetly.

*The biggest downside (which I only found after the migration) is, that Nextcloud currently has a [bug](https://github.com/nextcloud/server/issues/11826) which breaks encryption of the data, if the primary storage is an object store. But for me, the upsides of an object store overruled this problem.*

## Migrate the data
The very first step was to create a new bucket on amazon s3, as Nextcloud assumes exclusive usage over the whole bucket. They also decided to use a different layout for the files on object storage compared to a regular filesystem, which means I have to create a mapping between the data before copying the files.

On a regular filesystem Nextcloud stores the data in the data directory in the same folder structure as the files are saved by the user. On object storage the files are all located in the base directory and named after the file-ID in the database. To sync the data into the object store I adapted a script, which I found on [this blog article](https://pedal.me.uk/migrating-a-nextcloud-instance-to-amazon-s3/) to create symlinks to every file with the correct names. These symlinks will then be synced into the object store so that the Nextcloud will find the correct files at the correct locations.

The scripts use some variables, which have to be replaced before executing them:
* `<mysql_host>`: The hostname of your Nextcloud mysql database
* `<mysql_database>`: The name of the mysql database
* `<mysql_user>` and `<mysql_password>`: Credentials with readwrite access to the Nextcloud mysql database
* `<s3_bucket>`: The name of your s3 bucket
* `<s3_bucket_region>`: The region of your s3 bucket (e.g. `eu-central-1` for Frankfurt)
* `<aws_accesskey>` and `<aws_secretkey>`: Credentials with readwrite access to the s3 bucket
* `</path/to/installation>`: The base directory of your Nextcloud installation

Before I started migrating the data, I put the Nextcloud into maintenance mode to make sure, that no files are changed in the migration process

```
$ cd </path/to/installation>
$ sudo -u www-data php occ maintenance:mode --on
```

#### Step 1 - sync the data to s3
I used the following script to create a directory `s3_files` which contains correctly named symlinks to all files stored in the Nextcloud

```
#!/usr/bin/env bash
DATA_DIR='</path/to/installation>/data/'
DB_NAME='<mysql_database>'
DB_OPTIONS='--host=<mysql_host> --user=<mysql_user> --password=<mysql_password>'

mysql ${DB_OPTIONS} --batch --disable-column-names --database=${DB_NAME} << EOF > user_file_list.txt
   select concat('urn:oid:', fileid, ' ', '${DATA_DIR}', substring(id from 7), '/', path)
     from oc_filecache
     join oc_storages
      on storage = numeric_id
   where id like 'home::%'
   order by id;
EOF

# Get the meta files
mysql ${DB_OPTIONS} --batch --disable-column-names --database=${DB_NAME} << EOF > meta_file_list.txt
   select concat('urn:oid:', fileid, ' ', substring(id from 8), path)
     from oc_filecache
     join oc_storages
       on storage = numeric_id
     where id like 'local::%'
     order by id;
EOF

mkdir s3_files; cd s3_files

while read target source ; do
    if [ -f "$source" ] ; then
        ln -s "$source" "$target"
    fi
done < ../user_file_list.txt

while read target source ; do
    if [ -f "$source" ] ; then
        ln -s "$source" "$target"
    fi
done < ../meta_file_list.txt

cd ..; rm user_file_list.txt meta_file_list.txt
```

When the symlink creation is finished you should have a directoy with a lot of symlinks (about 60.000 in my case). If you check that the namings are correct, you should use `find` or the `-F` option of `ls` to not sort the files (otherwise it will be painfully slow). A regular `s3cmd sync` would've taken very long (about 5 days) to finish the data sync, so I created a small wrapper script, which uploads 40 files in parallel. This reduced the upload time to about 5 hours

```
#!/usr/bin/env bash
upload() {
  file=$(cut -d/ -f2 <<< $1)
  cmd="s3cmd put -F $file s3://<s3_bucket>/$file"
  echo `$cmd`
}
export -f upload

cd s3files
find . | parallel -j40 upload
```

After the upload was finished I checked if the data was correctly stored, which might not be the case if some connections failed during the upload. I used the following script to validate that filename and filesize are equal in both locations

```
#!/usr/local/bin/env bash
s3cmd ls s3://<s3_bucket>/ |awk '{print "$4 $3"}' | sort > files_s3.txt
for f in $(find s3_files/ -type l | cut -d/ -f2); do
  echo "s3://<s3_bucket>/$f $(ls -Ll s3_files/$f | awk '{print $5}')"
done | sort > files_local.txt

diff -ruN files_local.txt files_s3.txt
```

Files which were different I re-uploaded manually (using the above defined `upload()` function).

#### Step 2 - Update the database
If the upload was successful and there are no differences between the datasets, the database needs to be updated to reflect the new location. Please be aware, that up to this point no acutal changes were made to the Nextcloud instance. Changing the database wrongly *will* break your Nextcloud instance, so make sure that you really understand what you (or the scripts I provide) are doing.

I recommend to create a backup/dump of the database before the changes to have the possibility to roll them back

```
$ mysqldump --result-file=dump.sql <mysql_database>
$ mysql --database=<mysql_database> --host=<mysql_host> --user=<mysql_user> --password=<mysql_password>  <<EOF
   update oc_storages
      set id = concat('object::user:', substring(id from 7))
      where id like 'home::%';
   update oc_storages
     set id = 'object::store:amazon::<s3_bucket>'
     where id like 'local::%';
EOF
```

#### Step 3 - Configure the objectstore
When the database is correctly updated we need to finalize the migration by adding the object store to the Nextcloud configuration. Therefor edit the file `</path/to/installation>/config/config.json` and add the `objectstore' definition

```
  1 'objectstore' =>
  2 array (
  3   'class' => '\\OC\\Files\\ObjectStore\\S3',
  4   'arguments' =>
  5   array (
  6     'bucket' => '<s3_bucket>',
  7     'autocreate' => true,
  8     'key' => '<aws_accesskey>',
  9     'secret' => '<aws_secretkey>',
 10     'use_ssl' => true,
 11     'region' => '<s3_bucket_region>',
 12     'use_path_style' => false,
 13   ),
 14 ),
```

When the config is updated you can disable the maintenance mode

```
$ cd </path/to/installation>
$ sudo -u www-data php occ maintenance:mode --off
```

Your Nextcloud will now use the data stored in s3 and you can safely delete the old files in the `</path/to/installation>/data` directory.

***To be continued...***