# Installation #

All the examples below use the default installation paths. You are free to change these.

## Client ##

First you need to place the vsh script somewhere you can run it. If you are a single user on a single machine you can place it in ~/bin/ or if it is a shared machine, anywhere where all admins can access it like /usr/bin or any other folder in $PATH.

To set up a template and create your first user key simply run: vsh -t

```
# vsh -t
/home/user/.vsh/hosts created.

Please add vsh-hosts to this file.

ctstate-folder exist. Skipping.
Creating default keys:
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
/home/user/.vsh/keys/vsh-user-user generated

Distribute this to ~root/.ssh/authorized_keys on all vsh-hosts:

command="/opt/vsh/bin/vshd user=user" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDH7rWth7g7mlpV5BxDgry8Zj/WSg74HcsaqMCAlcfYqy+p7b6rv9KYOmfKKhArExHEnLhLs1tCmTMyAxnk08Aq57mUzUu8VnUiy0DZm6G65SnzucMkN9OWtqRp8f4hTNXLKm1WYUWc70HuxRF7kQxE6eTpCV8EJppKcRWNB08ICmZU+u1ZGyemVVkwDCTqNqg20XIQNyCGKOcBnu2Iiu2oHu9RBz/Dy7R1/61EQLQ8z5MEB+dYg4MJb+rZ3/TYhoV/2W/yW2P6bLW2Y10Jl2kLWjyPDpr5Ru6hNFk5vKNmw3dbhe3w1OhaD7sMgGX7Ae+Mm39r9HhkTOpACy4Xkgp1 vsh-user-user
```

If you feel that you need more roles, create more keys with `vsh -g <role>`.

Add hosts to `~/.vsh/hosts`:

```
# hosts file
vsh-host1	# You can put comments here
vsh-host2
#unused-vsh-host
```

Distribute keys with the suffix .dist.pub to the vsh-server (see below). If this is to be used in an environment with several admins you might want to use some kind of script that gather keys, check them and creates a authorized_keys file to be distributed by cfengine or puppet to your vsh-servers. 

## Server ##

To set up a vshd, download vsh and create some folders.

```
mkdir -p ~/sandbox
cd ~/sandbox
git clone git://github.com/yownas/vsh
mkdir -p /opt/vsh/{bin,etc,libexec}
```

Install `vshd` script and dependencies:

```
cd ~/sandbox/vsh
cp -p vshd /opt/vsh/bin/
cp -p libexec/* /opt/vsh/libexec
``` 

Create config file. **TODO:** Include sampe ini file in 
vsh distribution.

```
cat <<EOF >/opt/vsh/etc/vshd.ini
# Groups.
# Format: group:user1[[,user2],user3]
[groups]

# List of containers. Need to be fqdn of container or * (any)
# Format: fqdn:user1|@group1[,user2|@group2]
[containers]

# vsh hosts. fqdn or * (any)
# Format: fqdn:user1|@group1[,user2|@group2]
[hosts]

# Operations permissions
[operations]

# Module persmissions
[modules]
EOF
```


### vshd.ini ###

The example below we have three users, Alice, Bob and Eve.

Alice and Bob are members of the wheel-group and have access to all
containers and modules.
Alice is also a member of the admin-group, but has to use a separate key named "admin" to access that role. That group has access to actions as move-container (OpenVZ only) and start/stop. It also has access to all "hosts" which means that she is allowed to log in on the host running vshd.

Under [hosts] you can use * or a hostname. Typically you would use * but if you want to limit access to a subset of hosts you can set a hostname and let your configuration tool like cfengine or puppet push it out to all your hosts. Users will only be allowed to access hosts where both hostname and user/group matches.

The user Eve only has access to the container syslog.domain and will get logged in as the user "logmaster" instead of root.

Example `/opt/vsh/etc/vshd.ini`:

```
# Groups.
# Format: group:user1[[,user2],user3]
[groups]
wheel:alice,bob
admins:alice+admin

# List of containers. Need to be fqdn of container or * (any)
# Format: fqdn:user1|@group1[,user2|@group2]
[containers]
*:@wheel
syslog.domain:eve=logmaster

# vsh hosts. fqdn or * (any)
# Format: fqdn:user1|@group1[,user2|@group2]
[hosts]
*:@admins

# Operations permissions
[operations]
move:@admins
startstop:@admins

# Module persmissions
[modules]
*:@wheel
```

### proxy-clients ###

If you want to be able to ssh to hosts as root via vshd you can add them to /opt/vsh/etc/proxy-clients.

You should assume that the user "root" is used by default when loggin in. If you want to log in as another user simply add it to the hostname. (See below.)

The first field is an alias and can be used as an easy to remember name for hosts with long names ot hosts that can not be resolved with dns. It is also possible to use the alias field to create different ways to access a host. (See dbhost1 example below.)

One thing to keep in mind is that if you have several vshd-instances the aliases should all point to the same target, or the vsh-client will act unpredictably. 

You can also let vshd act as a proxy for vsh. Add "* vsh:" to proxy-clients and set up vsh as usual for the user running vshd (typically root). Also make sure that you have the vsh-script in `/opt/vsh/bin/`. (This feature is experimental but "should" work, how practical and/or safe it is could be questioned.)

Using "* vsh:" can be useful if you have hosts on a NATed network and only have one public IP on the outside. (Maybe. As I stated before, this is experimental and added mostly because "it was possible".)

Example `/opt/vsh/etc/proxy-clients`:
```
# <alias>	ssh:<hostname|fqdn|ip>
# *		vsh:
ns1.some.domain	ssh:ns1.some.domain
web		ssh:webserver.with.very.long.name.omg
gw		ssh:192.168.0.1
dbroot1		ssh:dbhost1
dbadmin1	ssh:mysql@dbhost1

*	vsh:
```

### User keys ###

To allow users to run vshd you need to add their vsh-keys to ~root/.ssh/authorized_keys. Make sure that all the keys has a proper command-option at the beginning and that the path to vshd is correct and that "user=<username>" is set to the user you expect.

Example below has one user with two keys, one with the default user-role and the second with the admin-role.

Example: `~root/.ssh/authorized_keys`:

```
command="/opt/vsh/bin/vshd user=alice" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDP59ep36DGG6A7UpGc4/YG/IbfwcCHsTLy5bNafAAsjBB09zk
BCArA8U7XCLbtY4fEpEFzPqhWBh8xVYC0Uh35rR/KTMTMKFacRul7t7iGiBvVb/bDOgOBqrSwgB0f2dYe8s0BEGCf3i3yJ1CP2TavoXbtaGCE8ionP7+6kAroSo1
rTquN4gCC1+tBslLUp7LzhgSgNmVKEQ2ra9ZM7GqTOiPP vsh-alice-user
command="/opt/vsh/bin/vshd user=alice+admin" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCzT9fH7Pka6C6rVdxcRT7GVM483bzDkmbyErz4GPKS2
HFeZNM/+CUGcps/ZBTnoqx6ncw9lF4lqnv1NT+mII940yqiEuLrqH01vReyWclWUJFIKDuX4q7XVFPkp059hhzZ4oGYLDLQYJaGqcmBFggSdJW17GxwvpQL1ew5D
x9+PffEqZ6/5Jfzm7TT9/IuKXOPDRWpivJYxrfQYvW92mIJaAm5 vsh-alice-admin
```

If you've come this far you should now be able to run vsh -l and see a list of containers (and proxied hosts). :)

### User key distribution ###

The security of vsh is based on how it uses ssh-keys for authentication. When you distribute the dist.pub-keys from users you MUST check a couple of things.

* The name after user= in `command="/opt/vsh/bin/vshd user=alice"` MUST be the same as the username that owns the file.
* The command in `command="/opt/vsh/bin/vshd user=alice"` MUST be `/opt/vsh/bin/vshd` unless you set another path.
* The command and argument in `command="/opt/vsh/bin/vshd user=alice"` MUST not be anything other than the path to vshd and the `user=<username>` argument.
* The file MUST be owned by the correct user and only writable by that user. Not group or world writable.
* The comment should also contain the name, but it is only a comment and not really needed.
* Do not add users to hosts that they do not need access to or that contain containers that they are not allowed to access.

If you have several users you can check if any of the keys are the same. They shouldn't be and could mean that someone is trying to give another user his/her privileges.

Users should be encouraged to create new keys and change periodically instead of just setting a new password on the old keys.

### Security ###

vsh and vshd are just shell-scripts, there is no guarantee that noone won't be able to get around the privileges setup in `vshd.ini`. Assume that anyone who has a vsh-key added to a host might gain admin access. To keep hosts and containers safe only add administrators you trust to your hosts and let less trustworthy admins access only a sub-set of hosts.

If you are using vshd with only proxy-clients, and no containers that need root-access, you can let vshd run as an unprivileged user by adding a username to the host in the hostlist. Just remember that you might need to change vshprefix=/opt/vsh to something your unprivileged user can access.

This also allows to set up separate vshd-instances on a single machine to let different groups have their own set of trusted keys and proxied clients they may access.
