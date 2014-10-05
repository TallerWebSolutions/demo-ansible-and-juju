Get started with Juju and Ansible at the same time
==================================================
Testing Ansible locally isn't a good option, so we can use Juju to create LXC containers and run a playbook into it. Also because you are in the Juju ecosystem you can deploy charms like MySQL and use it together.
This tutorial is only for learning purposes, for you to know the power of the two tools at the same time.

### Pre-requisities:
1. **Understand what a [Linux container](https://www.youtube.com/watch?v=_jBTHyo0mEQ) is**.
2. **GNU/Linux Ubuntu Trusty**.
If not, you can use the [Juju Vagrant flow](https://juju.ubuntu.com/docs/config-vagrant.html) and skip the next step #3.
3. **Juju local juju-local and bootstraped**.
If not, here's [how to install](https://juju.ubuntu.com/docs/config-LXC.html).
4. **Ansible**.
Here's [how to install](http://docs.ansible.com/intro_installation.html#latest-releases-via-apt-ubuntu).

### Objectives:
1. Install Apache2.
2. Install `php5-fpm` with `php-apc` and others `PHP-*`.
3. Create a virtualhost.
4. Install Composer, Drush and Git.

### Let's deploy our machine.
```
$ juju deploy cs:precise/ubuntu my-project
```
Wait till the machine get set, and then copy the IP.
You can keep watching the Juju or Linux containers status.
```
$ watch juju status
$ ## OR
$ sudo watch lxc-ls --fancy
```

### Creating our first project using Ansible.
```
$ mkdir my-project-servers
$ cd my-project-servers

$ ## Roles
$ mkdir roles

$ ## Playbook
$ touch playbook.yaml

$ ## Inventory
$ touch inventory
```

### Install Ansible roles needed for the project.
Recommended roles from [geerlingguy](https://galaxy.ansible.com/list#/users/219).
```
$ ansible-galaxy install -n -p ./roles geerlingguy.apache geerlingguy.php geerlingguy.composer geerlingguy.drush geerlingguy.git
```
You should have the 5 roles in the `roles` directory.

### Relate the server to the playbook using an inventory.
Probably your machine is ready! so let's get the IP.
```
$ juju status
...
services:
...
  my-project:
    charm: cs:precise/ubuntu-4
    exposed: false
    units:
      my-project/0:
        agent-state: started
        agent-version: 1.20.7.1
        machine: "2"
        public-address: 10.0.3.191 <-- o/ HERE !!
```
Put the content below in the `inventory` file:
```
[webserver]
10.0.3.191  ansible_ssh_user=ubuntu
```

### Create the `playbook.yaml` file.
Look with attention to the comments (#) and pay attention to the indentation.
```yaml
---
- hosts: webserver

  # Every command is going to be executed with sudo.
  sudo: yes

  # Override global variables.
  vars:
    www_owner: ubuntu
    www_group: www-data

  # Execute tasks before anything else.
  pre_tasks:
    - name: "Adding www's owner user in the web server group."
      user: >
        name={{ www_owner }} groups={{ www_group }}

    - name: "Set permissions for the /var/www directory."
      file: >
        path=/var/www mode=6775
        owner={{ www_owner }} group={{ www_group }}
        state=directory

    - name: "Create the Virtual host's dir and set permissions."
      file: >
        path=/var/www/my-project mode=6775
        owner={{ www_owner }} group={{ www_group }}
        state=directory

  # Execute the Ansible roles.
  roles:
    # Install GIT.
    - role: geerlingguy.git
      git_package:
        - git

    # Install and configure Apache2.
    - role: geerlingguy.apache
      # Create Virtual Hosts.
      apache_vhosts:
        # Additional properties: 'serveradmin, extra_parameters'.
        - servername: "my-project.local"
          documentroot: "/var/www/my-project"

    # Install and configure PHP with FPM.
    - role: geerlingguy.php
      php_memory_limit: "512M"
      php_max_execution_time: "60"
      php_upload_max_filesize: "64M"
      php_apc_enabled_in_ini: true
      php_apc_cache_by_default: "1"
      php_apc_shm_size: "96M"
      php_date_timezone: "America/Sao_Paulo"
      php_enable_webserver: true
      # Enable PHP-FPM.
      php_enable_php_fpm: true
      # Enable APC.
      php_enable_apc: true
      # Install the packages.
      php_packages:
        - php5
        - php5-mcrypt
        - php5-cli
        - php5-common
        - php5-curl
        - php5-dev
        - php5-fpm
        - php5-gd
        - php-pear
        - php5-mysql

    # Install Composer.
    - geerlingguy.composer

    # Install Drush (Drupal's shell).
    - role: geerlingguy.drush
      drush_version: 6.x
```

### Show time motherphokas !!
Run the playbook:
```
$ ansible-playbook -i inventory playbook.yaml
```
You should end up with _something_ like this at the end:
```
...
PLAY RECAP *************************************************************
10.0.3.191              : ok=37   changed=12   unreachable=0    failed=0
...
```
The important part is `failed=0`, if anything fails the process stop.

**NOTE:**
In case you get the following error:
```
...
TASK: [geerlingguy.php | Ensure php-fpm is started andenabled at boot (if configured).] ***
failed: [10.0.3.191] => {"failed": true}
msg: service not found: php-fpm

FATAL: all hosts have already failed -- aborting

PLAY RECAP **************************************************************
           to retry, use: --limit @/home/vagrant/playbook.yaml.retry

10.0.3.191               : ok=18   changed=11   unreachable=0    failed=1
...
```
Probably because the [pull request #13](https://github.com/geerlingguy/ansible-role-php/pull/13) wasn't accepted or fixed.
You can fix this by cloning the role from the fork:
```
$ mv roles/geerlingguy.php roles/geerlingguy.php.OLD
$ git clone git@github.com:TallerWebSolutions/ansible-role-php.git -b php-fpm-daemon-var roles/geerlingguy.php
$ # Remove the .git dir otherwise to avoid loosing the role.
$ rm -rf roles/geerlingguy.php/.git
```

### How to know if it works.
Access from your browser the IP (ex. [10.0.3.191](10.0.3.191)) of the container and you should get something like:
```
It works!

This is the default web page for this server.

The web server software is running but no content has been added, yet.
```
That means your default server is already up and running!! but what about the virtual host you created?

First, let's try to simulate the install of our application adding tasks at the end of the `playbook.yaml` file, pay attention at the indentation spaces.
```
...
  # Execute tasks after the roles of pre tasks.
  tasks:
    - name: "Create's the index.php"
      shell: >
        echo "<?php phpinfo(); ?>" > index.php
      args:
        chdir: /var/www/my-project
        creates: /var/www/my-project/index.php
      sudo_user: "{{ www_owner }}"
      notify:
        - restart apache
        - restart php-fpm
```
Run the playbook:
```
$ ansible-playbook -i inventory playbook.yaml
```
After the playbook successfully ends, you need to configure your `/etc/hosts` in order to resolve the domain `my-project.local`.
```
$ sudo echo "10.0.3.191  my-project.local" >> /etc/hosts
```
Access from your browser [my-project.local](http://my-project.local) and hopefully you'll get the famous `phpinfo()` page.

### Destroy everything and start from zero.
Destroy the `my-project` service and the machine, by knowing the number.
```
$ juju status
...
services:
...
  my-project:
    charm: cs:precise/ubuntu-4
    exposed: false
    units:
      my-project/0:
        agent-state: started
        agent-version: 1.20.7.1
        machine: "2"  <----------- o/ HERE !!
        public-address: 10.0.3.191
...
$ juju destroy-service my-project
$ juju destroy-machine 2
```
After the process is done, you can deploy a new machine again:
```
$ juju deploy cs:precise/ubuntu my-project
```
Edit the `inventory` and `/etc/hosts` files to update with the new IP, and then run the playbook to get a new web server for the project:
```
$ ansible-playbook -i inventory playbook.yaml
```

### You shouldn't be doing this for production use.
The Juju ecosystem have already some Ansbile charms, where you can take advantages from the `hooks based` architecture, using the `charm-helpers` which uses `tags` when running the `playbook.yaml` into the machine.
[Here's an example](https://github.com/sebas5384/charm-drupal) of an Ansible charm.