# Ansible roles for deploying eID Proxy components

This playbook contains roles for installing and configuring the SATOSA proxy, along with other tools surounding it.

It includes the roles:
* [debops.unattended_upgrades](#debopsunattended_upgrades)
* [debops.users](#debopsusers)
* [devsec.ssh](#devsecssh)
* [grnet.apt](#grnetapt)
* [grnet.base](#grnetbase)
* [grnet.certbot](#grnetcertbot)
* [grnet.core](#grnetcore)
* [grnet.docker](#grnetdocker)
* [grnet.ferm](#grnetferm)
* [grnet.logging](#grnetlogging)
* [grnet.merge_vars](#grnetmerge_vars)
* [grnet.openssl-certificate](#grnetopenssl-certificate)
* [grnet.puppet.base](#grnetpuppet.base)
* [grnet.satosa](#grnetsatosa)
* [nginxinc.nginx](#nginxincnginx)

This playbook has only been tested with Debian Stretch.

## Roles

### debops.unattended_upgrades
Installs and configures the unattended upgrades package.

### debop.users
Creates and configures a set of users.

### devsec.ssh
Configures the ssh and ssh keys.

### grnet.apt
Configures apt and install any required apt packages. All apt packages are installed using this role in conjunction with the [grnet.merge_vars](#grnetmerge_vars) role.

### grnet.base
Acts as a base role and is simply responsible of acting as a base that calls the roles: grnet.merge_vars, grnet.core, grnet.apt, debops.unattended_upgrades, debops.apt_listchanges, grnet.ferm, devsec.ssh, debops.users

### grnet.certbot
Installs certbot and creates the necessary certificate using the nginx server or certbots own webserver.

### grnet.core
Configures root user, ansible user and sets timezone.

### grnet.docker
Installs docker. This role uses [grnet.merge_vars](#grnetmerge_vars) and [grnet.apt](#grnetapt) in order to add the docker apt repo and install the docker package.

### grnet.ferm
Creates ferm rules. Ferm rules are created using this role in conjunction with the [grnet.merge_vars](#grnetmerge_vars) role.

### grnet.logging
Uses rsyslogd and logrotate to handle the logs of an application writing on syslog. This roles is usually included inside another role, more information on the available variables can be found [here](roles/grnet.logging/README.md)

### grnet.merge_vars
A hacky role which takes advantage of the reflection properties of the ansible variable system. This role is used to collect and merge multiple lists or dictionaries into one based on their names.

### grnet.openssl-certificate
Creates a self-signed openssl certificate. More information on the available variables can be found [here](grnet.roles/openssl-certificate/README.md)

### grnet.satosa
Installs and configures SATOSA. More information on the available variables can be found [here](roles/grnet.satosa/README.md)

### nginxinc.nginx
Installs and configures nginx.  This role is taken from [nginxinc.nginx](https://github.com/nginxinc/ansible-role-nginx/tree/af54ab1401717161aa37783bf2ed71f0d28d714d)

### Groups
Generally we use 3 groups: proxy, nginx and satosa.

nginx and satosa are responsible for installing the corresponding component.
The proxy group is used to install both nginx and satosa, since most of the time they are run at the same host.

### Variables
In each inventory several variables are defined to override the default values.
In the `group_vars/all` folder basic variables are configured. In the ferm_defs file we define ferm variables that are used in the ferm rules (all ansible variables of the format `{role_name}__ferm__def_variables_to_merge` will be used to define ferm definitions). In the users file we define the users that will be created and configured using the `debops.users` role.

Other than that we define the necessary group_vars and host_vars for each use case.

### Usage
In order to run this ansible playbook we must:
1. Define the necessary inventory and variables. For this example let's assume we want to create a SATOSA installation connected to EIDAS so we will have an nginx receiving all requests, the SSL will be terminated at nginx then the request will be redirected to SATOSA. An example inventory achieving this is `inventories/EIDAS`.
2. After defining the necessary variables and hosts we must run the playbook located at `play/bootstrap.yml` which will create the necessary users, since our ansible roles are configured to as the ansible user and not as debian which is the default user of okeanos' debian VMs (of course this could be changed). So we must run:
`ansible-playbook plays/bootstrap.yml -i inventories/EIDAS`
3. After this is run we must run the playbook located in `site.yml`, this playbook is simply used to run several the other playbooks which will install the requested components, all playbooks except `site.yml` are located in the plays folder. So we must run:
`ansible-playbook site.yml -i inventories/EIDAS`

After this out installation must be complete. For further information about configuring the playbook look to our [docs](docs/eidas-deployment.md)
