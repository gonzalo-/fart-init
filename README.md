```
  /$$$$$$                      /$$          /$$           /$$   /$$    
 /$$__  $$                    | $$         |__/          |__/  | $$    
| $$  \__//$$$$$$   /$$$$$$  /$$$$$$        /$$ /$$$$$$$  /$$ /$$$$$$  
| $$$$   |____  $$ /$$__  $$|_  $$_//$$$$$$| $$| $$__  $$| $$|_  $$_/  
| $$_/    /$$$$$$$| $$  \__/  | $$ |______/| $$| $$  \ $$| $$  | $$    
| $$     /$$__  $$| $$        | $$ /$$     | $$| $$  | $$| $$  | $$ /$$
| $$    |  $$$$$$$| $$        |  $$$$/     | $$| $$  | $$| $$  |  $$$$/
|__/     \_______/|__/         \___/       |__/|__/  |__/|__/   \___/  

```

## What is it?

fart-init it's a little script that try to get cloud-init files (meta-data, user-data and network-data)
to configure the server with it, after that, it will clean up the server and itself. The idea is to
create a new and clean server from a template [proxmox/qemu](https://x61.sh/log/2023/05/17052023102313-qemu_proxmox_openbsd_template.html) with a
basic configuration.

## What can it do?

fart-init can do:

- set a main user and ssh-keys
- set a root password
- set the network (dhcp or static)
- install a list of packages (soon)

what it can't do:

- it won't resize or change partitions on your VM
- configure any service on base system (you should do this with something else)

what it will do by default for you:

- enable unwind(8) as resolver
- disable sndiod(8) since I asume this is a server
- clean up all packages installed
- delete all residual files in the server that are not standard
- delete the user given as main and created it again

## Setup

I based fart-init on cloud-init's files, so for it, you will need 3 files meta-data, user-data and network-data, each one
of them has the information that it will be extract by fart-init to set up the server.

You can give these 3 files in 2 different ways to fart-init:

- Over the cloud-init option in Proxmox (VM -> Cloud-init -> Edit -> Regenerate)
- Over a webserver reacheable from the VM

*** CAVEATS: Proxmox is a linux server and their password hashing is different from the one we use in OpenBSD, so the password given
over Proxmox for the user won't work, use the ssh-key instead and set up a root password inside fart-init, or just use the default one
for root which is "fart-init"

If you will serve the files by Proxmox just fill the "Cloud-init" information as you want and regenerate it, that should be enough, if you
want to serve the files over a webserver, they should look like this:

```
$ cat user-data
#cloud-config
hostname: fart-init
username: gonzalo
password: puffy01
ssh-key: "ssh-ed25519 AAAAC3NzaFsTghZaSAAIPxVebz+gL0DqbsikzlMBA0SM059VkOmEGly3b24SnNH"
chpasswd:
  expire: False
```

```
$ cat meta-data
instance-id: 999/fart-init
local-hostname: fart-init
```

network-data - dhcp
```
$ cat network-data
version: 1
config:
    - type: physical
      name: eth0
      mac_address: '16:70:0d:8e:dd:a4'
      subnets:
      - type: dhcp4
    - type: nameserver
      address:
      - '9.9.9.9'
      search:
      - 'fart.home'
```
network-data - static
```
$ cat network-data
version: 1
config:
    - type: physical
      name: eth0
      mac_address: '16:70:0d:8e:dd:a4'
      subnets:
      - type: static
        address: '192.168.0.211'
        netmask: '255.255.255.0'
        gateway: '192.168.0.1'
    - type: nameserver
      address:
      - '192.168.0.123'
      search:
      - 'fart.home'
```

As I said you can serve these files with a webserver (httpd(8)) or just using python for example like:

```
proxmox:~/files# ls -al
total 20
drwxr-xr-x 2 root root 4096 May 22 19:12 .
drwx------ 5 root root 4096 May 22 15:54 ..
-rw-r--r-- 1 root root   53 May 22 15:58 meta-data
-rw-r--r-- 1 root root  345 May 22 18:54 network-config
-rw-r--r-- 1 root root  186 May 22 19:12 user-data
proxmox:~/files# python3 -m http.server --directory .
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Adjust the line `FART_SRV="10.0.2.2:8000"` inside fart-init to your needs and be sure the port is open on your firewall.

## How to use it?

After created your [OpenBSD template](https://x61.sh/log/2023/05/17052023102313-qemu_proxmox_openbsd_template.html), you need to download
fart-init and give it permissions, something like this (assuming your proxmox or virtual network is 10.0.2.0/24):

```
vm_template# cd /usr/local/sbin/
vm_template# ftp -V http://10.0.2.2:8000/fart-init
vm_template# chmod 755 fart-init
vm_template# echo '/usr/local/sbin/fart-init 2>&1 | tee /var/log/fart-init.log' > /etc/rc.local
```

And that is pretty much of it, keep in mind this is a very early version of it and it could be fail, I ran it several times on my setup without
issues but this could change in yours.

During the first booting process you will see something like this:

![alt text](https://github.com/gonzalo-/fart-init/blob/main/imgs/fart-init_booting.png?raw=true)

And on the second boot:

![alt text](https://github.com/gonzalo-/fart-init/blob/main/imgs/fart-init_booted.png?raw=true)

Have fun!
