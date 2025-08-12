---
date:
  created: 2024-12-07
tags:
  - linux
  - vm
  - security
  - szte
draft: true
---

# Some exploits I have noticed in SZTE's Iriny classrooms

Do not use these for anything, please.

<!-- more -->

## Some context

Since I am studing at SZTE's TTIK, I had a few classes in the Iriny cabinet. They have a few classrooms full of PCs. These run some kind of Linux as a hypervisor, the actual classwork is done on VMs running on these PCs in VirtualBox. The disks inside these VMs is reset after shutdown so no permanent files can be created (in theory).

## The exploits

I found out about these exploits on my own, however I never used them for anything. The cabinet's terms state that experimenting with gaining privileges is a punishable offence. These are so fundamental errors in configuration that I wouldn't count my actions as *experimentation*, but I know that this won't save me if they ever come after me.

### 1. Docker, my beloved

The guest user was in the *docker* group and could start containers with privilleged access. That's it. You *could* start a container with root rights and do anything you want. You can even mount the host's *sudoers* file and add the guest user or edit any file for that matter.

``` docker-compose
# an example docker-compose file
services:
  exploit:
    image: debian:bookworm-slim
    volumes:
      - /etc/sudoers:/etc/sudoers
      - /etc/passwd:/etc/passwd
    privileged: true # the good sauce
    network_mode: host
```

![docker, my beloved](https://media1.tenor.com/m/GZJayzTKO1MAAAAd/docker-heart-locket.gif)

Since then, they updated their VMs and forgot to add the guest user to the *docker* group, so now docker is installed, but you can't use it. :facepalm:

No problem. We will get our docker access back another way.

### 2. Mount

Since the VMs reset after every shutdown, you need a place to store your projects. For that reason they created a script that mounts the users NFS share after authentication.

I looked inside the script. It calls mount using sudo. So we know that the user has access to mounting NFS shares, surely that's all. I checked the sudoers file and I was dissapointed. The guest user has access to mount without restrictions. :facepalm:

GNU/Linux is an amazing system with a well designed feature rich filesystem. One of these amazing features is loopbacks and binding. You can mount a folder over another esentially overwriting its contents. This means you can mount a new *sudoers.d* folder with custom rules over */etc/sudoers.d*. So you just create a new folder, make a file with you rules and mount it over ther original folder. You need to set the persmission on the bind using some flags when using mount, it's simple as that.

``` bash
#!/bin/bash

LOCAL="/home.local/valaki/sudoers.d"
mkdir -p "$LOCAL"
echo "valaki ALL = NOPASSWD:ALL" > "$LOCAL"/all
sudo chown -R root:root "$LOCAL"
sudo mount -o bind,uid=0,gid=0 "$LOCAL" "/etc/sudoers.d"
sudo -i
sudo umount "/etc/sudoers.d"
```

You just gained root access inside the VM. You can run docker again. :smile:

### 3. VirtualBox

This was a real bad one, but I think they fixed it since then.

Basically VirtualBox captured some key combinations like ++ctrl+p++. ++ctrl+s++ opens the currently running VMs settings. :facepalm: From there you could have changed networking settings or mounted a folder from the host. I didn't test this, but you might have been able to mount the *sudores* or *shadow* file and edit it from the VM.

### 4. Scanning

All the PCs in the building are on the same network and have routing setup between the classrooms. Every host and every VM has its own hostname generated from a pattern (eg. pc22505-03 -> room 225, PC 5, VM 3 (Windows 10)).

Using a simple bash script you can scan the whole building end determin which PCs are online and what VM they are running.

In theory you could ssh into one of these VMs and use it as a proxy to bypass CooSpace's IP restriction on exams, but ssh is not allowed for the guest user (but we saw how to edit it earlier, so you might be able to).

## Conclusion

These exploits are not really dangerous on their own apart from the VirtualBox one. However, I didn't exactly put a lot of effort into finding these. So what if I did, how many vulnerabilities could I find?

Also there is a server for the students that you can ssh into. It runs the same Debian VM that the PCs run, so same exploits.
