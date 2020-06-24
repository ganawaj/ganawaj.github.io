---
layout: post
title:  Starting multiple docker-compose containers from Systemd easily
categories: [Docker, Docker-Compose]
excerpt: "In my homelab, I run a lot of different containers and one things I didn't want to do was write a seperate systemd unit 
file for each container, especially if the container is just for testing or temporary. It also became really tedious
to write a unit file, reload systemd and possibly delete it later every time. So I found a way to make that a lot easier."
---

I ran across this [Gist](https://gist.github.com/mosquito/b23e1c1e5723a7fd9e6568e5cf91180f) from 
[Mosquito](https://github.com/mosquito) that eventually led me to this 
[article](https://community.hetzner.com/tutorials/docker-compose-as-systemd-service) at Hetzner. 


```yaml
[Unit]
Description=docker-compose %i service
Requires=docker.service network-online.target
After=docker.service network-online.target

[Service]
WorkingDirectory=/etc/docker-compose/%i
Type=simple
TimeoutStartSec=15min
Restart=always

ExecStartPre=/usr/local/bin/docker-compose pull --quiet --ignore-pull-failures
ExecStartPre=/usr/local/bin/docker-compose build --pull

ExecStart=/usr/local/bin/docker-compose up --remove-orphans

ExecStop=/usr/local/bin/docker-compose down --remove-orphans

ExecReload=/usr/local/bin/docker-compose pull --quiet --ignore-pull-failures
ExecReload=/usr/local/bin/docker-compose build --pull

[Install]
WantedBy=multi-user.target
```

Two great things about this unit file is that it always keeps your container images up to date and will build containers
if your docker-compose.yml is configured to do so. 

### Getting Started 

Simply create a directory that will store your docker-compose files. For me, I created a ZFS filesystem mounted 
at /docker-compose 

```
zfs create pool0/files/file0/media/docker-compose
zfs set mountpoint=/docker-compose pool0/files/file0/media/docker-compose
```

Or simply create the directory with mkdir:

```
mkdir /docker-compose
```

### Systemd File

Next, let's create the unit file: 
```
vi /etc/systemd/system/docker-compose@.service
```

Find this line: `WorkingDirectory=/etc/docker-compose/%i` and change it to the directory you created.
In my case, `WorkingDirectory=/docker-compose/%i`

Another thing to keep in mind is where your docker-compose executable is stored. If it's different than `/usr/local/bin/`
you'll need to update this in the unit file. 

Finally, reload the service file
```
systemctl daemon-reload
```

### Container/Service

Create the directory for your new service. I will be creating a Unifi Controller

```
mkdir /docker-compose/unifi-controller
```
The name you use for this directory is what you'll use to start, stop and reload your service. It doesn't need to match
the name of the service your creating, it can be literally anything.

Drop your docker-compose.yml file in this directory, and your'e ready to start your container. 

Simply run:
 ```
 systemctl start docker-compose@[directory name]
``` 
or in my case:
```
systemctl start docker-compose@unifi-controller
```
