---
title: 'Get your logs together'
author: 'Christian Beneke'
published: true
date: '03-02-2016 00:00'
taxonomy:
    tag:
        - Server
---

Everyone in charge of at least a small virtualized server instance knows the struggle of debugging some software and grepping through the logs in "thousands" of locations.
A friend of mine told me about [Graylog](https://www.graylog.org/) lately and so I decided to get a small instance of that  piece of software up and running on my root server.

# Getting started
As one always should, I started reading the [docs](http://docs.graylog.org/en/2.0/) and was a little stunned by the dependencies... *as im not that much of a friend with java...*
But still, I gave it a try!

As the minimal setup consists of at least one elasticsearch node, one graylog2-server, one instance of mongodb, the graylog2-web-interface and *something* feeding the servers logs with data. I decided to use our database virtual machine for the mongodb, the web machine for the webinterface and one more powerful machine for the elasticsearch and graylog2-server instance.

# Setting up the services
As we're running [Arch Linux](https://www.archlinux.org/) on our virtual machines I didn't even have to bother thinking about wheter or not to use the .deb or .rpm files. So I took the long and interesting path of the *Manual Installation*.
Thankfully the documentation is really good and you also don't have much to do getting the services running.

## Mongodb
Install the latest package of mongodb, run the mongod-service and you are ready to go. If you want you can set up a user et cetera et cetera.. but just for testing, this was too much overhead for me either. Make sure, that your mongodb is just accessable from the machines which *must* be able to connect to the database!

## The graylog2-server and elasticsearch
After the database, I started with the backbone of the whole Graylog instance. To fulfill the dependencies I installed *jre8-openjdk-headless* from the official repsoitories and *elasticsearch-1.7.3-1* from the archive (as *elasticsearch 2.x* is not yet supported). Afterwards it was just getting and extracting the files:

    cd /opt
    wget https://.../graylog-1.3.3.tgz     (1)
    tar xf graylog-1.3.3.tgz
    mkdir -p /etc/graylog/server
    cp graylog.conf.example                \
       /etc/graylog/server/server.conf
<small>(1) https://packages.graylog2.org/releases/graylog2-server/graylog-1.3.3.tgz  </small>

Setting up a *passwort_secret* (`pwgen -N1 -s 96`), a *root_password_sha2* (`echo -n yourpassword |shasum -a 256`), the *rest_listen_uri* and the *mongodb_uri* in the config file is enough to finish the basic configuration. You can of course change multiple other things, but see for yourself.

Elasticsearch was even more easy to setup: Install the package and start the service. You can - as I did - change the cluster name (don't forget to tell Graylog the correct name too!) and tune the service within its config file, but it's not necessary for small instances.

You can then start up your graylog2-server instance: `/opt/graylog2-1.3.3/bin/graylogctl start`

## The web-interface
A service analyzing your logs is kind of boring when you can't see what it's doing - isn't it? Also the latter configuration (which propably can also be done with the graylogctl tool) may be easier.
So I sat up the web-interface next: The installation of the web-interface is almost the same as the graylog2-server service, except you have just java as dependency and need to download the [web-interface tar archive](https://packages.graylog2.org/releases/graylog2-web-interface/graylog-web-interface-1.3.3.tgz) from the [download page](https://packages.graylog2.org/releases/graylog2-web-interface/graylog-web-interface-1.3.3.tgz). When extracted you must just edit the */opt/graylog-web-interface-1.3.3/conf/graylog-web-interface.conf* and setup the *graylog2-server.uris* to your graylog2-server's IP and port, as well as an *secret key* (like the one in your server's configuration). Then you are ready to run the web-interface with `/opt/graylog-web-interface-1.3.3/bin/graylog-web-interface`.

# Feeding the server with logs
To get actual data into your Graylog instance you can use the so called collectors. Therefor you can download the [latest stable build](https://github.com/Graylog2/collector#binary-download) and extract the files kinda like already done with the server and web-interface (again assuming the installation to */opt*). You then must edit the */opt/graylog-collector-0.4.2/config/collector.conf* (which you can copy from the *collector.conf.example*).

The config file has 3 parts: the configuration, the inputs-block and the output-block.
In the configuration you must set the *server-url* matching your graylog2-server's address.

For the inputs-block you must add one block for each logging input. These blocks look like

    $inputID {
      type = "$type"
      $options
    }

where the inputID must be unique among all configured inputs (Else the latter one will overwrite the first). E.g. I configured an exim-mail and exim-rejected block:

    exim-mail {
      type = "file"
      path = "/var/log/exim/mainlog"
    }
    exim-reject {
      type = "file"
      path = "/var/log/exim/rejectlog"
    }

For detailed information for the options, have a look at the [installation guide](http://docs.graylog.org/en/2.0/pages/collector.html#configuration).

The outputs block contains one or more output destinations. You can for example set up an GELF (**G**raylog **E**xtended **L**og **F**ormat) Input for the server on the web-interface and use this input as an output for your collector. For myself I defined an default input and used this for the collector:

    gelf-tcp {
      type = "gelf"
      host = "$hostname"
      port = 12201
      client-tls = false
    }

With one or more inputs and output the collector is then ready to run.

# Systemd service files
As it's bad practice and annoying to always start the graylog-services by hand, I searched for some systemd service files, to handle the services. Following [hadret's git repo](https://github.com/hadret/scripts-graylog2) I then wrote three simple scripts to start the server, the web-interface and the collector which can be found in [my git repo](https://github.com/sattelite/scripts-graylog2).

# Summary
The whole project seems to be very well written, but for small servers it is hardly overpowered. My setup consists of a 4GB RAM VM and two 2GB RAM VMs, which is enough to get it running... Not sure how well it will scale though.
The installation is straight forward and easy, when you stick to the docs and know what you are doing.

I'm going to test it a little while and will add a longtime report including the topics *How did it help me with my workflow*, *How to set it up with Ansible* and *Which features I like most*.

---

# Advanced usage
As it can be seen in the following picture, Graylog is very "clusterable" and seems to be suitable for very large instances.
![sketch of a large setup with Graylog](http://docs.graylog.org/en/2.0/_images/architec_bigger_setup.png)

Please refer to the [homepage](https://www.graylog.org) and [docs](http://docs.graylog.org/en/2.0) of Graylog for further information.