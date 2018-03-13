# SaltStack Security Best Practices

SaltStack is an amazing configuration management tool, but without adhering to general best practices and security principals it can quickly become a security nightmare. This document will provide some general principals for using SaltStack securely, and is divided into sections specific to the minion, the master, and common principals that apply to both.

Securing the [Master](#securing-the-master)

Securing [Minions](#securing-minions)

## Security Configuration for Minion and Master

### Changing Salt User Privileges
[SaltStack Documentation Link](https://docs.saltstack.com/en/latest/ref/configuration/nonroot.html)

By default Salt masters and minions run as a root user. This can be altered on either the master or the minion by changing the `user` parameter in their configuration files but comes with subtle drawbacks. 

Specifically:
* Running a **Salt master** as an unprivileged user does not change its capability to access minions as the user they are running as. (example: if a minion is running salt as root, commands given by an unprivileged master will still run as root on the minion) 
* Running a **Salt minion** as an unprivileged user will prevent salt from doing things like installing packages or modifying users unless special access controls *(ie sudo)* are given to the minion user.

Because of these drawbacks it is normal to allow Salt to run as root except in specific use cases. 
ha
#### If it is necessary to change the Salt user on either the minion or the master: 

Uncomment and modify the `user:` parameter inside of the respective master or minion configuration files located at:

Master: `/etc/salt/master` 

Minion: `/etc/salt/minion`

Then restart Salt with `systemctl restart salt-minion.service` or `systemctl restart salt-master.service`

## Securing the Master

Because of the master's ability to run *usually as root* on connected minions, hardening of the master is very important. The master should be secured in the way of any other high priority target.

Because of the nature of a Master/Minion relationship, hardening the Salt-Master from unwanted access is necessary to ensure the integrity of its minions. 

Generally, the Salt-Master should be an independent system with tightly controlled access. The Salt-Master should also not be exposed to WAN.

### Access Control to the Master
[SaltStack Documentation Link](https://docs.saltstack.com/en/latest/topics/eauth/access_control.html)

The most important security measure to take when establishing a master is implementing a well defined and maintained method of access control. This at a minimum should include individual user accounts on the master that can be accessed via ssh. If the master is accessed remotely, it is advised to place it behind a vpn or at a minimum to disable password access over ssh and require a valid key. 

Salt also contains many features that can be used to restrict which salt commands specific users, or any user can, can run.

On the opensource version of SaltStack, the following methods of configuring user permissions are allowed:

**publisher_acl:** This is useful for allowing local system users to run Salt commands without giving them root access. If you can log into the Salt master directly, then publisher_acl allows you to use Salt without root privileges. If the local system is configured to authenticate against a remote system, like LDAP or Active Directory, then publisher_acl will interact with the remote system transparently.

**external_auth:** This is useful for salt-api or for making your own scripts that use Salt's Python API. It can be used at the CLI (with the -a flag) but it is more cumbersome as there are more steps involved. The only time it is useful at the CLI is when the local system is not configured to authenticate against an external service but you still want Salt to authenticate against an external service.

In SaltStack Enterprise, user controls can be modified by a GUI, and allows for the benefit of specific role and scope based permissions. The SaltStack Enterprise access controls have the added benefit of combining access to the master with user specific permissions.

### Using publisher_acl

The publisher ACL system is configured in a configuration file on the Salt master with the `publisher_acl:` option. This can either be on the `/etc/salt/master`or in a .config file located in `/etc/salt/master.d/<file>`. 

Where the `publisher_acl` option will specify what is allowed on a given master, the `publisher_acl_blacklist` will specify only what is disallowed.

An example of publisher_acl config that would be found in on a master:
```YAML

publisher_acl:
  # Allow thatch to execute anything.
  thatch:
    - .*
  # Allow fred to use test and pkg, but only on "web*" minions.
  fred:
    - web*:
      - test.*
      - pkg.*
  # Allow admin and managers to use saltutil module functions
  admin|manager_.*:
    - saltutil.*
  # Allow users to use only my_mod functions on "web*" minions with specific arguments.
  user_.*:
    - web*:
      - 'my_mod.*':
          args:
            - 'a.*'
            - 'b.*'
          kwargs:
            'kwa': 'kwa.*'
            'kwb': 'kwb'
```

In the above example, 'thatch' has been given access to run any command on the master, where 'fred' can only run specific commands on minions named 'web*'.

The following directories are required for publisher_acl must be modified to be readable by the users specified:

`chmod 755 /var/cache/salt /var/cache/salt/master /var/cache/salt/master/jobs /var/run/salt /var/run/salt/master`

### Using external_auth

Salt's External Authentication System (eAuth) allows for Salt to pass through command authorization to any external authentication system, such as PAM or LDAP.

Configuring eAuth is very similar to publisher_acl, and can be done by editing the master configuration file.

An example configuration of eAuth using the PAM Salt module:

```YAML
external_auth:
  pam:
    thatch:
      - 'web*':
        - test.*
        - network.*
    steve|admin.*:
      - .*
```

## Securing Minions
Communication between a master and minions is always TLS encrypted. Therefore, a compromised minion will not be able to snoop on an other minion's communication with their master.

Because of this, the main security vulnerability involving minions becomes a compromised minion possessing privileged information that it should not have, or gaining access to that information once it is compromised.

Both of these problems can be easily avoided by proper handling of secrets via pillars.

### Handling Secrets on Minions
[SaltStack Documentation Link](https://docs.saltstack.com/en/latest/topics/best_practices.html#storing-secure-data)

The most important aspect of handling secrets with Salt is that they should always be contained inside a pillar file, and not in a state file. This is because in a Master/Minion Salt configuration, all state files are distributed to all minions, regardless of grains.

Example:

`/srv/salt/mysql/user.sls`

```YAML

include:
  - mysql.testerdb

testdb_user:
  mysql_user.present:
    - name: frank
    - password: "test3rdb"
    - host: localhost
    - require:
      - sls: mysql.testerdb
```
In the previous state file, the mysql password for frank would be accessible by any minion.

Instead, the password should be stored in a pillar file like so:

`/srv/pillar/mysql.sls`

```YAML

mysql:
  lookup:
    name: testerdb
    password: test3rdb
    user: frank
    host: localhost
```

This pillar file can now be referenced inside our previous state file. When this state is run on a minion, the minion must request the pillar data from the master as it is ran.

New State File:

`/srv/salt/mysql/user.sls`

```YAML

include:
  - mysql.testerdb

testdb_user:
  mysql_user.present:
    - name: {{ salt['pillar.get']('mysql:lookup:user') }}
    - password: {{ salt['pillar.get']('mysql:lookup:password') }}
    - host: {{ salt['pillar.get']('mysql:lookup:host') }}
    - require:
      - sls: mysql.testerdb
```

### Properly targeting minions with grains 
State and pillar files will target specific minions using grains.

Basic info about grains can be found [here](https://docs.saltstack.com/en/latest/topics/grains/index.html).

Because grains can be set by users with access to minion configuration files, targeting minions with grains should only be done when using `grains['id']` which is the Minion ID. This is because if the minion ID of a system changes, it's public key must be re-accepted by the master.

[SaltStack Documentation Link](https://docs.saltstack.com/en/latest/faq.html#faq-grain-security)