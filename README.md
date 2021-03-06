<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Install dokuwiki](#install-dokuwiki)
- [NOPASSWD for the session](#nopasswd-for-the-session)
- [First sight](#first-sight)
- [Update the system](#update-the-system)
- [Prepare to use ansible](#prepare-to-use-ansible)
    - [Get ansible](#get-ansible)
    - [Get our plays](#get-our-plays)
    - [Get dependencies](#get-dependencies)
    - [Add an inventory](#add-an-inventory)
- [Use our plays](#use-our-plays)
    - [Apache](#apache)
    - [PHP](#php)
    - [Dokuwiki](#dokuwiki)
    - [Letsencrypt](#letsencrypt)
- [Backup](#backup)

<!-- markdown-toc end -->

# Install dokuwiki

A journal of dokuwiki installation

```
export domain=domain.tld
```

# NOPASSWD for the session

```
echo '%sudo-nopasswd ALL = (ALL:ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/sudo-nopasswd
sudo addgroup --system sudo-nopasswd
sudo adduser $USER sudo-nopasswd
newgrp - sudo-nopasswd
export USER=$(id -un)
# Don't forget to "sudo deluser $USER sudo-nopasswd" at session end
```

# First sight

After `ssh node.$domain -AX`

```
w

lsb_release -a
dpkg -l | wc -l

cat /proc/meminfo | head

sudo apt install lshw
sudo lshw -short

# sudo apt install util-linux
lscpu
lsblk

sudo iptables-save

df -h
ps -auxwwf | less

/sbin/ifconfig
/sbin/route
ss -tarp

awk NF /etc/ssh/sshd_config | grep -v ^#

awk -F: '$3 > 1000' /etc/passwd

sudo find / -path /proc -prune -o -perm -4000
sudo find / -path /proc -prune -o -size +50M

last
sudo lastb | awk '{print$1}' | sort | uniq -c | sort -nr | head
```

# Update the system

```
sudo apt update
sudo apt upgrade
cat /proc/version
ls -l /boot | grep vmlinuz | head -1
# We sould reboot
```

# Prepare to use ansible

## Get ansible

For the dependencies

```
sudo apt install ansible
```

Get the latest stable version

```
sudo apt install git
sudo mkdir -p /usr/local/ext; sudo chown $USER:staff /usr/local/ext; sudo chmod g+w /usr/local/ext
sudo adduser $USER staff
newgrp - staff
export USER=$(id -un)
(cd /usr/local/ext && test -d ansible-stable-2.9 || git clone --branch stable-2.9 --recursive git://github.com/ansible/ansible.git ansible-stable-2.9)
```

Note that in real life we would be on a control node, not on an install node

## Get our plays

```
git clone git@github.com:thydel/install-dokuwiki.git
```

## Get dependencies

```
cd install-dokuwiki
export ANSIBLE_ROLES_PATH=~/.ansible/roles
mkdir -p $ANSIBLE_ROLES_PATH
ansible-galaxy install -r requirement.yml
```

## Add an inventory

```
echo wiki ansible_host=localhost ansible_connection=local domain=$domain > inventory
```

# Use our plays

## Apache

A very basic invocation of `geerlingguy.apache`

```
export ANSIBLE_INVENTORY=inventory
source /usr/local/ext/ansible-stable-2.9/hacking/env-setup -q
ansible-playbook install-apache.yml
sudo apt install lynx
wget http://localhost -qO- | lynx -stdin -dump | head
wget http://wiki.$domain -qO- | lynx -stdin -dump | head
```

## PHP

A minimal invocation of `geerlingguy.apache`

```
ansible-playbook install-php.yml -e test=1
wget http://localhost/info.php -qO- | lynx -stdin -dump | head
sudo rm /var/www/html/info.php
```

## Dokuwiki

- Choose to use [petermosmans/dokuwiki][] galaxy role but sime simple patches needed, so
- Fork [PeterMosmans/ansible-role-dokuwiki][] as [thydel/ansible-role-dokuwiki][]
- [Uses new apt way and corrects src var for unarchive][]
- [Configures for a vhost][]

Role arguments comme from role README with
- Addition of a `vhost` param
- Change of default values for `user`, `provision` and `configure_apache2` param
- Change of `admin` user password hash (`hash=$(htpasswd -bnBC 10 "" $password | tr -d ':\n')`)

```
ansible-playbook install-dokuwiki.yml
wget http://wiki.$domain -qO- | lynx -stdin -dump | head
```

[petermosmans/dokuwiki]:
	https://galaxy.ansible.com/petermosmans/dokuwiki "galaxy.ansible.com"

[PeterMosmans/ansible-role-dokuwiki]:
	https://github.com/PeterMosmans/ansible-role-dokuwiki "github.com repo"

[thydel/ansible-role-dokuwiki]:
	https://github.com/thydel/ansible-role-dokuwiki "forked github.com repo"

[Uses new apt way and corrects src var for unarchive]:
	https://github.com/thydel/ansible-role-dokuwiki/commit/09accc8d759078038de3460d037d2e62dba5d170 "github.com commit"

[Configures for a vhost]:
	https://github.com/thydel/ansible-role-dokuwiki/commit/a871ce1dbc57e901e23a99697a1546cc330d2dbf "github.com commit"

## Letsencrypt

As my previous experience with letsencrypt.org was limited to
obtaining an renewing a unique wildcard certificate, I hesitate between
using a role based on `certbot` and the native `acme` ansible module
set.

I tried and failed to find a convincing existing role for `certbot`
but it seems such a role would be overkill for this oneliner task
anyway.

```
ansible-playbook install-certbot.yml
wget -q --server-response http://wiki.$domain
wget https://wiki.$domain -qO- | lynx -stdin -dump | head
```

Note that as `certbot` do configure the TLS vhost replaying
`install-dokuwiki.yml` would mangle the configuration, but playing
again `install-certbot.yml` will would fix it.

# Backup

- Minimal 7 days TGZ of the whole filetree
- Should be remote sync
- Could be crypted
- Could use simple rsync base incremental (link based) scheme
- Should be generated by a play using common data with other plays for file paths

```
sudo mkdir -p /usr/local/var/backups
echo '30 6 * * *  root tar -C /var/www/vhosts -czf /usr/local/var/backups/wiki.'$domain'.$(date +\%u).tgz wiki.'$domain | sudo tee -a /etc/cron.d/wiki_$(echo $domain | tr . _)
```
