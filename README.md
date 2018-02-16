# Ansible scripts to provision and deploy Odoo

These are [Ansible](http://docs.ansible.com/ansible/) playbooks (scripts) for managing an [Odoo](https://github.com/odoo/odoo) server.

## Requirements

You will need Ansible on your machine to run the playbooks.
These playbooks will install the PostgreSQL database, NodeJS and Python virtualenv to manage python packages.

It has currently been tested on **Ubuntu 16.04 Xenial (64 bit)**.


Install dependencies running:
```
ansible-galaxy install -r requirements.yml
```

## Bash scripts

### Default User
`script/default_user.yml`

Reads local SSH key and passes it to `create_user.sh` executed in the host with SSH root connection.
You can define an env var `SSH_PATH` if your SSH key is stored in another path other than the default `~/.ssh/id_rsa.pub`

### Create User
`script/create_user.yml`

Creates the default_user `odoo` and copies the SSH key (first argument) in authorized keys of the user.
Changes the SSH root login permissions.

## Playbooks

### Sysadmins
`sysadmins.yml` - Creates default user `odoo` and sysadmins defined in your `inventory/host_vars/YOUR_HOST/conf.yml` in a dictionary called `sysadmins`.

The structure to declare user is:

```yml
# inventory/host_vars/<YOUR_HOST>/config.yml

sysadmins:
  sysadmin1:
    key: "{{ lookup('env', 'HOME') }}/.ssh/id_rsa.pub"
    state: present
  sysadmin2:
    key: ../pub_keys/sysadmin2.pub
    state: present
```

Where `sysadmin1` and `sysadmin2` are the name of sysadmins users in the server.

Use `state: absent` to remove a user.

After executing this playbook the `odoo`'s authorized SSH keys is removed. To log in to the server as `odoo` user you should login with your sysadmin user and execute `sudo su odoo`.
All other users must belong to `odoo` group to manage the system service.

TASKS:
- Create default_user
- Create all sysadmin
- Add ssh keys
- Add sudo permisses

### Provision
`provision.yml` - Installs and configures all required software on the server.

The structure to create users is similar that `sysadmins`:

```yml
# inventory/host_vars/<YOUR_HOST>/config.yml

users:
  user1:
    key: ../pub_keys/user1.pub
    state: present
```

Where `user1` is your username and the name os your SSH key file.

TASKS:
- Users and Groups management
- Create directories structure
- Install common packages
- Install PostgreSQL database and create a user
- Install NodeJS and LESS

### Deploy
`deploy.yml` - Deploys source code from Odoo Nightly and installs Python requirements.

TASKS:
- Install and create VirtualEnv
- Ansistrano deploy:
  - Download the source code
  - Before link task: Build
  - Before link task: Install requirements.txt
- Add systemd service

### Deploy Custom Modules
`deploy_custom_modules.yml` - Deploys the custom or thirdy part modules that you need.

You can make a repository with submodules pointing your module repository. [Like in this example](https://github.com/danypr92/odoo-organization-custom-modules)

Put custom modules repository url in your `inventory/host_vars/your_host/config.yml` file:

```yml
#inventory/host_vars/<YOUR_HOST>/config.yml

custom_modules_repo: https://github.com/danypr92/odoo-organization-custom-modules.git
custom:modules_repo_branch: master
```

TASKS:
- Ansistrano git deploy.
- Update odoo.service to add addons.

## Roles

### Sysadmin

- Creates default user `odoo`
- Creates sysadmins with permissions

### Common

- Creates users (developers)
- Installs system packages
- Creates folder structure and configures permissions
- Creates virtualenv
- Installs Postgres and NodeJS
- Adds service unit
    
### Odoo Config

- Creates Odoo configuration file
- Restarts Odoo service

# Installation instructions

For the first playbook (sysadmin.yml) is needed have a `odoo` user with your SSH pub key.

If you not have this user but have acces like root, you can use the `default_user.sh` script.

#### Script to create default user.

Execute the `script/default_user.sh` to create the `odoo` user and add your SSH key.

System state:
- Permit Root SSH login (modify `/etc/ssh/sshd_config`)
- Access without password (copy your SSH key)

### Step 1 - SysAdmins

The **first time** thet execute this playbook use the user `odoo`

`ansible-playbook playbooks/sysadmins.yml -u odoo`

All the next times use your personal sysadmin user:

`ansible-playbook playbooks/sysadmins.yml -u USER`

USER --> Your sysadmin user name.

### Step 2 - Provision

`ansible-playbook playbooks/provision.yml -u USER`

USER --> Your sysadmin user name.

### Step 3 - Deploy

`ansible-playbook playbooks/deploy.yml -u USER`

USER --> Your user name (not need be sysadmin)

### Step 4 - Deploy custom modules

`ansible-playbook playbooks/deploy_custom_modules.yml -u USER`

USER --> Your user name (not need be sysadmin)

## System Administration

### Default User `odoo`

Used to execute `odoo.service` and is a a superuser.

### Sysadmins

The sysadmins are the superusers of the environment.
They have `sudo` access without password for all commands.

**They can execute `sysadmins.yml`, `provision.yml`, `deploy.yml` and `deploy_custom_modules.yml` playbooks.**

### Users (Developers)

They are users without `sudo` permissions.
They are in the `odoo` group and can execute the next commands withous password in `sudo` mode:

```
sudo systemctl start odoo.service
sudo systemctl stop odoo.service
sudo systemctl status odoo.service
sudo systemctl restart odoo.service
```

**They can execute `deploy.yml` and `deploy_custom_modules.yml` playbooks.**

## DB Admin Password

Add password to the `super admin` that manage the dbs.
In `inventory/host_vars/host_name/secrets.yml` add the key `admin_passwd` to protect the creation and management of dbs.
`secrets.yml` is a encrypted vault. Run `ansible-vault edit secrets.yml` to change this password.

# Development

In the development environment (`local.odoo.net`) you must use the sysadmin user `odoo`.

`ssh odoo@local.odoo.net`

## Using LXC Containers

In order to run the `scripts/create-contqainer.sh` script, you need install [LXC](https://linuxcontainers.org/).

The script in `scripts/create-container.sh` will help you to create a development environment using LXC containers.

The script will:

* Create container
* Mount your project directory into container in `/opt/<project_name>`
* Add container IP to `/etc/hosts`
* Create `odoo` group with same `gid` of project directory
* Create `odoo` user with same `uid` and `gid` of project directory
* Add system user's SSH public key to `odoo` user
* Install python2.7 in container
* Run `sys_admins.yml` playbook

When the execution ends, you have a container ready to provision and deploy the app.

__You can find the configuration variables in `scripts/confg/lxc.cfg`.__
