---
title: Things to do on a fresh linux installation
date: 2022-08-30 12:00:00 +0330
categories: [Linux, Ubuntu]
tags: [linux-preparation]     ## TAG names should always be lowercase
---
## Create new sudo user
first we create the user
```bash
sudo adduser madman
```
then add it to sudoers
```bash
sudo usermod -aG sudo madman
```
now we check if the user can run all commands
```bash
sudo -l -U madman
```

## Disable root login

we disable root logins with this command
```bash
sudo passwd -l root 
```

This will lock the password for the root user and you won’t be able to access the root account with its password until a new one is set.

## Update and Upgrade the system

```bash
sudo apt-get update -y && sudo apt-get upgrade -y
```
and reboot the server (important).

## Configure Automatic Upgrades

first install ```unattended-upgrades```

```bash
sudo apt-get install unattended-upgrades
```

then we reconfigure it
```bash
sudo dpkg reconfigure --priority=low unattended-upgrades
```

then we can check the config file
```bash
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
``` 

## Install SSH server & configure key-based authentication

### Install ssh server
we use ```openssh-server``` as our ssh-server
```bash
sudo apt-get install openssh-server
```
### configure key based authentication
now we have to create an ssh key-pair on both the server and client machine.
to do that we have to run this command:
```bash
ssh-keygen
```

basically the key based authentication works by copying the client's ssh public key to the server's authorized_keys file.

to do this the easy way we just run this simple command:
```bash
ssh-copy-id <user>@<server-address>
```
this will automatically copy the client's public key to the server's trusted keys over ssh connection.

if we want to do this manually we have to copy the contents of the client's public key to the server's authorized_keys file.

on the client machine:
```bash
cat ~/.ssh/id_rsa.pub
```
and we copy the output.
then on the server:
```bash
nano ~/.ssh/authorized_keys
```
paste and we're done.

### Disable password based authentication
now that we have enabled key-based authentication it's logical that we disable password-based authentication for extra security.

To do this have to edit the ssh daemon configuration file. 

```bash
sudo nano /etc/ssh/sshd_config
```

Uncomment these 2 fields and set them to no:
```
PasswordAuthentication no
ChallengeResponseAuthentication no
```
also to disable root account login:
```
PermitRootLogin no
```
now we restart ssh daemon
```bash
sudo systemctl restart sshd
```

## Configure Static IP
```bash
sudo nano /etc/netplan/01-netcfg.yaml
```
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
     dhcp4: no
     addresses:
        - 192.168.0.222/24
     gateway4: 192.168.0.1
     nameservers:
       addresses: [192.168.0.4]
```
apply the settings
```bash
sudo netplan apply
```

## hostname
check hostname:
```bash
hostnamectl
```
change hostname to ```rocketship```
```bash
sudo hostnamectl set-hostname rocketship
```
must also change in this file:
```bash
sudo nano /etc/hosts
```
## Timezone

check timezone:
```bash
timedatectl
```

change the timezone:
```bash
sudo timedatectl set-timezone America/Chicago
```

change with menu:
```bash
sudo dpkg-reconfigure tzdata 
```

## Firewall
allow outgoing traffic by default:
```bash
sudo ufw default allow outgoing
```
deny incoming traffic by default:
```bash
sudo ufw default deny incoming
```
allow ssh service:
```bash
sudo ufw allow ssh
```
enable the firewall:
```bash
sudo ufw enable
```
check status:
```bash
sudo ufw status
```
