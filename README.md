# Ansible scripts to provision and deploy Expenses

These are [Ansible](http://docs.ansible.com/ansible/) playbooks (scripts) for managing an [Expenses](https://github.com/chdelucia/expenses) server.

## Requirements

You will need Ansible on your machine to run the playbooks.
These playbooks will install `NodeJS` and `npm`.

It has currently been tested on **Ubuntu 16.04 Xenial (64 bit)**.


Install dependencies running:
```
ansible-galaxy install -r requirements.yml
```

## Playbooks

### sys_admins.yml

This playbook uses the community role [sys-admins-role](https://github.com/coopdevs/sys-admins-role).

This playbook will prepare the host to allow access to all the system administrators.

```yaml
# playbooks/my_playbook.yml
- name: Create all the users for system administration
  roles:
    - role: coopdevs.sys-admins-role
      vars:
        sys_admin_group: sysadmin
        sys_admins: "{{ system_administrators }}"
```

In each environment (`dev`, `staging`, `production`) we can find the list of users that will be created as system administrators.

We use `host_vars` to declare per environment variables:
```yaml
# inventory/host_vars/<YOUR_HOST>/config.yml

system_administrators:
  - name: pepe
    ssh_key: "../pub_keys/pepe.pub"
    state: present
    - name: paco
    ssh_key: "../pub_keys/paco.pub"
    state: present
```

The first time you run it against a brand new host you need to run it as `root` user.
You'll also need passwordless SSH access to the `root` user.
```
ansible-playbook playbooks/sys_admins.yml --limit=<environment_name> -u root
```

For the following executions, the script will asssume that your user is included in the system administrators list for the given host.

For example in the case of `development` environment the script will assume that the user that is running it is included in the system administrators [list](https://github.com/coopdevs/timeoverflow-provisioning/blob/master/inventory/host_vars/local.timeoverflow.org/config.yml#L5) for that environment.

To run the playbook as a system administrator just use the following command:
```
ansible-playbook playbooks/sys_admins.yml --limit=dev
```
Ansible will try to connect to the host using the system user. If your user as a system administrator is different than your local system user please run this playbook with the correct user using the `-u` flag.
```
ansible-playbook playbooks/sys_admins.yml --limit=dev -u <username>
```

### Provision
`provision.yml` - Installs and configures all required software on the server.

TASKS:
- Install security tools
- Install NodeJS and npm
- Install NGINX

# Installation instructions

### Step 1 - sys_admins

The **first time** thet execute this playbook use the user `root`

`ansible-playbook playbooks/sysadmins.yml --limit <environment_name> -u root`

All the next times use your personal system administrator user:

`ansible-playbook playbooks/sysadmins.yml --limit <environment_name> -u USER`

USER --> Your sysadmin user name.

### Step 2 - Provision

`ansible-playbook playbooks/provision.yml -u USER`

USER --> Your sysadmin user name.

## System Administration

### Default User `expenses`

### System Administrators

The sysadmins are the superusers of the environment.
They have `sudo` access without password for all commands.

**They can execute `sys_admins.yml`, `provision.yml`**

# Development

In the development environment (`local.expenses.org`) you must use the sysadmin user `expenses`.

`ssh expenses@local.expenses.org`

## Using LXC Containers

In order to create a development environment, we use the [devenv](https://github.com/coopdevs/devenv) tool. It uses [LXC](https://linuxcontainers.org/) to isolate the development environment.

This tool search a configuration dotfile in the project path. You must create a `.devenv`  file with the next declared variables:

```yaml
# expenses-provisioning/.devenv
NAME="expenses"
DISTRIBUTION="ubuntu"
RELEASE="xenial"
ARCH="amd64"
HOST="local.$NAME.coop"
PROJECT_NAME="expenses"
PROJECT_PATH="${PWD%/*}/$PROJECT_NAME"
DEVENV_USER="expenses"
DEVENV_GROUP="expenses"
```

You can modify them to change the environment.

Devenv will:

* Create container
* Mount your project directory into container in `/opt/<project_name>`
* Add container IP to `/etc/hosts`
* Install python2.7 in container
* Run `sys_admins.yml` playbook

When the execution ends, you have a container ready to execute sys_admins playbook, provision and deploy the app.
