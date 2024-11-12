<h1 align="center">System Automation with Ansible on Red Hat Enterprise Linux</h1>

## Introduction
Welcome to this course on system automation with Ansible on Red Hat Enterprise Linux. This course is designed to prepare you for the Red Hat Certified Engineer (RHCE) exam!

## Table of Contents
- [Ansible and setup Environment](#i-ansible-and-setup-environement)
- [Ad-Hoc Commands in Ansible](#ii-ad-hoc-commands-in-ansible)
- [Playbooks](#iii-playbooks)
- [Variables and Ansible Facts](iv-#variables-and-ansible-facts)
- [Ansible Vault](#v-ansible-vault)
- [Conditions & Loops](#vi-conditions--loops)
- [Jinja2 & Templates](#vii-jinja2--templates)
- [Task Handlers & Error Handlers](#viii-task-handlers--error-handlers)
- [Ansible Galaxy](#ix-ansible-galaxy)
- [Roles & RHEL System Roles](#x-roles--rhel-system-roles)
- [Storage Management](#xi-storage-management)

## I. Ansible and setup environement
Ansible is an automation tool that simplifies the management of servers and applications. It is "agentless," meaning it does not require special software to be installed on the machines you manage. It is popular because it is simple to use, efficient, and uses YAML files that are easy to understand.

### Ansible Architecture
- `Ansible Server` :  
This is the computer (laptop, PC, or server) where Ansible is installed. It contains:
- `Playbooks` :  
These are files that define what you want to do (such as installing software or configuring settings).
- `Inventory` :  
A list of servers you manage, called "hosts." It also helps to organize these servers into groups to simplify task management and execution.
- `ansible.cfg` : Contains the default configuration for Ansible.

### Playbook:
- `Roles` :  
Groups of tasks organized for a specific function (e.g., installing a web server).
- `Tasks` :  
These are the individual commands you want to execute (like "install nginx").

### Passwordless SSH:
Ansible communicates with target servers using SSH, without requiring a password each time.

### How Does It Work?
- You create playbooks and an inventory on your local machine.
- You establish a passwordless SSH connection to the target servers.
- Ansible gathers information (gather facts) about the target servers to understand their state.
- Ansible sends instructions (playbooks) to the servers.
- The servers execute the tasks defined in the playbooks.

### Configuration Steps

### 1. Prerequisites
Before starting, make sure you have:
- 3 virtual machines (1 control node & 2 target nodes) with Red Hat Enterprise Linux or CentOS installed.
[Download RHEL 9 VM](https://www.mediafire.com/file/mjs6r2sb7oqrqzk/RedHat_9.1.1_64-bit.zip/file)
- Python3 package installed on all 3 machines.

```bash
dnf install python3
```
### 2. Installer Ansible sous le control node.
```bash
dnf install ansible-core
```

### 3. Ajouter et configurer le 'remote user' sous les 3 machines.
```bash
useradd ansible
passwd ansible
```

```bash
vim /etc/sudoers.d/ansible
ansible     ALL=(ALL) 	NOPASSWD: ALL
```
Note: Before starting, you can name the machines as follows.
```bash
vim /etc/hosts
@ip control-node1
@ip target-node1
@ip target-node2 
```

### 4. Configure passwordless SSH connection from the control node as the remote user to the 2 target machines.
```bash
ssh ansible@localhost
ssh-keygen
ls .ssh
ssh-copy-id ansible@target-node1
ssh-copy-id ansible@target-node2
```
### 5. Add the inventory file on the control node.
```bash
[group1]
target-node1

[group2]
target-node2
```
### 6. Add the ansible.cfg file on the control node.
```bash
[defaults]
remote_user=ansible
inventory=/home/ansible/inventory
[privilege_escalation]
become=true
```

### 7. Test the passwordless SSH connection as the remote user.
```bash
ssh ansible@target-node1
```

### 8. Test the Ansible connection.
```bash
ansible target-node1 -m ping
```

## II. Ad-Hoc Commands in Ansible
### Syntax
`ansible <hosts> -m <module> -a <arguments> -i <inventory> [options]`
- `<hosts>`: The hosts on which the command should be executed. This can be "all", a group of hosts defined in your inventory file, or a specific IP address/hostname.
  
- `-m <module>`: The Ansible module to use for executing the task. For example, the "ping", "shell", "command", "copy", etc. module.
  
**Note**: To list all the modules
``` bash
ansible-doc -l
```
**Note**: To get the documentation for a module, you can use:
``` bash
ansible-doc module_name

ansible-doc -l | grep user
```
- `-a <arguments>`: The arguments to pass to the specified module. The way arguments are specified depends on the module being used.

- `-i <inventory>`: The inventory file that contains the list of hosts and their groups. If you don't use this option, Ansible will use the default inventory file located in `/etc/ansible/hosts`.

[options]: Other additional options can be specified, such as `-u <user>` to define the SSH user to use, `-b` to escalate privileges (sudo), etc.

## III. Playbooks

- The playbook extension can be `.yml` or `.yaml`.
- To check the syntax, use the following command:
``` bash
 ansible-playbook --syntax-check
 ```
- To execute the playbook:
``` bash
 ansible-playbook playbook.yml
 ```
 ### Syntaxe
```bash
- name: Description of the playbook
  hosts: [hôtes]
  become: [true/false]
  tasks:
    - name: Description of the task 1
      [module]: 
        [arguments]
    - name: Description of the task 2
      [module]: 
        [arguments]
```
### Examples

#### Exp 1
``` bash
- name: add alice, modify bob, delete charlie
  hosts: all
  become: true
  tasks:
  - name: add alice
    user:
      name: alice
      home: /home/alice
      shell: /bin/bash
      state: present
  - name: modify bob
    user:
      name: bob
      uid: 2006
      home: /home/bob
      shell: /bin/sh
      state: present
  - name: delete charlie
    user:
      name: charlie
      state: absent
```    

#### Exp 2
``` bash
- name: install and start service httpd
  hosts: all
  become: true
  tasks:
  - name: install httpd
    yum:
      name: httpd
      state: latest
  - name: start httpd
    service:
      name: httpd
      state: started
      enabled: yes
```

### playbook avec plusieurs plays
An Ansible playbook can contain multiple "plays". Each "play" applies a set of tasks to a group of hosts. This allows you to structure the playbook to perform different phases of a configuration or deployment.  

**Example:**  
- **play1:** Add a user  
- **play2:** Modify the user

Note: We use multiple plays so that we can execute the plays on different hosts `hosts`.

```bash
- name: play1 to add group and its user
  hosts: all
  become: true
  tasks:
  - name: add the group
    group:
      name: group1
      gid: 4900
  - name: add the user
    user:
      name: user1
      uid: 4008
      group: group1
- name: play2 to modify the user1
  hosts: all
  become: true
  tasks:
  - name: modify
    user:
      name: user1
      uid: 4007
```

## IV. Variables and Ansible Facts

### Variables
#### Using `vars`
In this method, we define variables directly in the playbook under the `vars` section.
Example:
```bash
- name: Ajouter utilisateur avec des variables définies
  hosts: all
  become: true
  vars:
    username: "alice"
    home_directory: "/home/alice"
    user_shell: "/bin/bash"
  tasks:
    - name: Ajouter utilisateur
      user:
        name: "{{ username }}"
        home: "{{ home_directory }}"
        shell: "{{ user_shell }}"
        state: present
```

#### Using `vars_files`
In this method, we store the variables in a separate file and then reference that file in the `vars_files` section.
```bash
vim variables.yml
```
```bash
username: "alice"
home_directory: "/home/alice"
user_shell: "/bin/bash"

```

```bash
vim playbook.yml
```
```bash
- name: Ajouter utilisateur avec un fichier de variables
  hosts: all
  become: true
  vars_files:
    - variables.yml
  tasks:
    - name: Ajouter utilisateur
      user:
        name: "{{ username }}"
        home: "{{ home_directory }}"
        shell: "{{ user_shell }}"
        state: present
```

### Facts
To display facts, use the following command:
```bash 
ansible node1 -m setup | less
```
### Filtering Facts Using Variables
To filter facts using variables, you can use a playbook task with the `debug` module to display specific facts. For example:
```bash 
ansible node1 -m setup -a "filter=ansible_default_ipv4"
```

Example1: Playbook for displaying facts.
```bash 
- name: Ansible Facts Playbook
  hosts: all
  tasks:
  - name: Display ansible fact
    debug:
      var: ansible_hostname
```

Example2: Playbook for displaying facts.
```bash 
- name: Ansible Facts Playbook
  hosts: all
  tasks:
  - name: Display Facts method1
    debug:
      var: ansible_default_ipv4.address
  - name: Display Facts method2
    debug:
      var: ansible_facts['default_ipv4']['address']
```
## V. Ansible-vault
`Ansible Vault` is a tool used to encrypt sensitive files, such as files containing passwords or API keys. It helps secure sensitive information within playbooks and other configuration files. Here is a detailed explanation of how to use Ansible Vault and the associated commands.

### Ansible Vault Commands
```bash
ansible-vault --help
```

`ansible-vault create`  
### Create a new encrypted file with Ansible Vault:
```bash
ansible-vault create protected_file.yml
```
After executing this command, you will be prompted to enter a password. This password will be used to encrypt the file. Once the file is created, you can view it to verify that it is indeed encrypted.
```bash
cat protected_file.yml
```

`ansible-vault view`  
This command allows you to view the content of an encrypted file without modifying it.
```bash
ansible-vault view protected_file.yml
```
You will be prompted to enter the password to decrypt and display the content of the file.

`ansible-vault edit`  
This command allows you to edit an encrypted file.
```bash
ansible-vault edit protected_file.yml
```
You will be prompted to enter the password, then the file will open in your default text editor.  

`ansible-vault rekey`  
This command changes the password of an encrypted file.
```bash
ansible-vault rekey protected_file.yml
```
You will be prompted to enter the old password, then a new password.

### Executing a Playbook with Encrypted Files
To run a playbook containing encrypted files, you can use the `--ask-vault-password` or `--vault-password-file` options.  

#### Option `--ask-vault-password`  
This option prompts you to enter the password to decrypt the file.
```bash
ansible-playbook playbook.yml --ask-vault-password
```

#### Option `--vault-password-file`  
This option uses a file containing the password to decrypt the file.
```bash
ansible-playbook playbook.yml --vault-password-file vault_password
```

Example:
1. Use Ansible Vault to create an encrypted password file containing the variable `password` and its value.
2. Create an unencrypted file containing the variable `username` and its value.
3. Write an Ansible playbook that uses these two files to create a user with the specified username and password.

```bash
ansible-vault create password.yml
password: "Password"
```

```bash
vim user.yml
username: "alice"
```

```bash
vim playbook.yml

- name: Créer un utilisateur avec un mot de passe chiffré
  hosts: all
  become: true
  vars_files:
    - user.yml
    - password.yml
  tasks:
  - name: Créer l'utilisateur
    user:
      name: "{{ username }}"
      password: "{{ password | password_hash('sha512') }}"
      state: present
```

```bash
ansible-playbook create_user.yml --ask-vault-password
```

Note:
`{{ password | password_hash('sha512') }}` ensures that the password stored and used in the system is secure and hashed, rather than storing and using the password in plain text.  
You can verify the hashed password by checking the `/etc/shadow` file.

## VI. Conditions & Loops

### Loop in Ansible
Ansible allows iteration over a list of items using loops. This simplifies repetitive tasks. Here is an explanation with concrete examples.
- `Simple loop with an embedded list`: Iterates directly over a list defined in the playbook.
- `Simple loop with a variable file`: Iterates over a list defined in an external variable file.
- `Loop with a dictionary`: Iterates over a list of dictionaries, allowing access to multiple values associated with each item.
#### Example 1: Simple loop with an embedded list
```bash
- hosts: all
  tasks:
    - name: afficher la liste
      shell: echo "{{ item }}"
      loop:
        - "one"
        - "two"
        - "three"
```
**NB:**
- `item`: This is a special variable used inside the loop to represent the current item in the list.
- `loop`: Contains the list of items to iterate over.

#### Example 2: Simple loop with a variable file
```bash
vim simple_loop.yml
```

```bash
numbers:
  - "one"
  - "two"
  - "three"
```

```bash
vim playbook.yml
```

```bash
- hosts: all
  vars_files:
    - simple_loop.yml
  tasks:
    - name: afficher la liste
      shell: echo "{{ item }}"
      loop: "{{ numbers }}"
```
Note: The playbook uses the list `numbers` defined in `simple_loop.yml`.

#### Example 3: Loop with a dictionary
```bash
vim dict_loop.yml
```
```bash
users:
  - username: user1
    his_group: group1
  - username: user2
    his_group: group2
  - username: user3
    his_group: group3
```

```bash
vim playbook.yml
```
```bash
- hosts: all
  vars_files:
    - dict_loop.yml
  tasks:
    - name: afficher la liste
      debug:
        msg: this is the user "{{ item.username }}" and his group is "{{ item.his_group }}"
      loop: "{{ users }}"
```
Note:
- `users`: Contains a list of dictionaries, each dictionary having the keys `username` and `his_group`.
- `loop: "{{ users }}"`: Uses the list `users` defined in `dict_loop.yml`.
- `item.username` and `item.his_group`: Accesses the values of the current dictionary in the loop.

### When in Ansible
The `when` condition in Ansible allows tasks to be executed only if certain conditions are met. If the condition is not met, the task is skipped, and Ansible indicates this action in the output as `skipping`.

#### Example Playbook with when:
```bash
- name: Install vim in CentOS and Ubuntu nodes
  hosts: web
  tasks:
    - name: Install vim on Centos
      yum:
        name: vim
        state: present
      when: ansible_facts['distribution'] == "CentOS"

    - name: Install vim on Ubuntu
      apt:
        name: vim
        state: present
      when: ansible_facts['distribution'] == "Ubuntu"
```
## VII. Jinja2 & Template
### Jinja2 Syntax and Template Module
Jinja2 is a templating engine used in Ansible to dynamically generate files. It allows you to include variables, loops, and conditions in configuration files using files with the `.j2` extension.

#### Basic Jinja2 Syntax:
- `Variables`: Used to insert dynamic values.
```bash 
{{ variable_name }}

```

- `Loops`: Used to repeat a section of code.
```bash 
{% for item in list %}
{{ item }}
{% endfor %}
```

- `Conditions`: Used to insert conditional code sections.
```bash 
{% if condition %}
Some content
{% endif %}
```

#### Template Module:
The `template` module in Ansible uses Jinja2 to create files based on templates. It allows copying a template file to a location on the remote host, replacing the variables with their current values.
Example of the template module:

```bash 
- name: Render a configuration file from template
  template:
    src: /path/to/template.j2
    dest: /path/to/destination/file
```

## VIII. Task handlers & Error handlers

### ignore_errors and ignore_unreachable in Ansible
`ignore_errors` is used to ignore errors on a specific task and continue executing the subsequent tasks.
```bash
- hosts: all
  tasks:
    - name: This task will fail but continue
      command: /bin/false
      ignore_errors: yes

    - name: This task will still run
      command: /bin/echo "Hello, World!"
```
In this example, even if the first task fails, Ansible continues and executes the second task.

`ignore_unreachable` is used to ignore unreachable hosts and continue executing tasks on other hosts. It is typically used at the playbook or task block level.
```bash
- hosts: all
  ignore_unreachable: yes
  tasks:
    - name: This task will run even if some hosts are unreachable
      command: /bin/echo "Running on reachable hosts"

```
In this example, if some hosts are unreachable, Ansible continues to execute the tasks on the reachable hosts. 

### Using Blocks (block)  
Blocks are used to group multiple tasks and apply common directives or conditions to them.  
Example of a block with a condition:
```bash
- hosts: all
  tasks:
    - block:
        - name: Install package
          yum:
            name: httpd
            state: present

        - name: Start service
          service:
            name: httpd
            state: started
      when: ansible_os_family == "RedHat"
```
Note: A block can be used for a condition containing multiple tasks.  

### Using block and rescue  
Blocks can be used to handle errors by using `rescue`, which defines the tasks to be executed in case of an error within the `block`.  

Simple example with `block` and `rescue`:
```bash
- hosts: localhost
  tasks:
    - block:
        - name: Run a command that may fail
          command: /bin/false

        - name: This task will be skipped if the above fails
          command: /bin/echo "This won't run"
      rescue:
        - name: Handle the error
          debug:
            msg: "There was an error, handling it now"
```
In this example:
The tasks inside the block are executed first.  
If a task inside the block fails, the tasks in the rescue section are executed to handle the error.  

### Using block and always
Blocks can also be used to execute final tasks, regardless of whether previous tasks succeed or fail, by using `always`.  

Simple example with `block` and `always`:
```bash
- hosts: localhost
  tasks:
    - block:
        - name: Run a command that may fail
          command: /bin/false

        - name: This task will be skipped if the above fails
          command: /bin/echo "This won't run"
      always:
        - name: Always run this task
          debug:
            msg: "This task always runs, regardless of the previous results"
```
In this example:
The tasks inside the `block` are executed first.  
The tasks inside the `always` are executed regardless of any errors in the block.  

### Using block, rescue, and always  
Blocks can also be used to handle errors with `rescue` and to execute final tasks with `always`.  

Simple example with `rescue` and `always`:
```bash
- hosts: localhost
  tasks:
    - block:
        - name: Run a command that may fail
          command: /bin/false

        - name: This task will be skipped if the above fails
          command: /bin/echo "This won't run"
      rescue:
        - name: Handle the error
          debug:
            msg: "There was an error"
      always:
        - name: Always run this task
          debug:
            msg: "This always runs"

```

In this example:

- The tasks inside the block are executed first.  
- If a task inside the block fails, the tasks in the rescue are executed to handle the error.  
- The tasks in the always are executed, regardless of whether the tasks in the block or rescue succeed or fail.

## IX. Ansible Galax
`ansible-galaxy` is a command-line tool for managing Ansible roles and collections. It allows you to download, install, and manage roles from public or private repositories.

### Installing a Role
``` bash
ansible-config dump | grep -i galaxy_server
```
To install a role from Ansible Galaxy, use the `ansible-galaxy install` command.  
For example, to install the role `geerlingguy.apache`, you can run:
```bash
ansible-galaxy install geerlingguy.apache
```
Note:  
By default, roles installed with `ansible-galaxy` are stored in the `.ansible/roles` directory. (Check with `ansible --version`)

### Change the Default Role Location
``` bash
ansible-config dump | grep -i roles
```
You can change the default location where `ansible-galaxy` installs roles by using the `ansible.cfg` configuration file.  
Add this line in the `ansible.cfg` file:
``` bash
roles_path = /path/to/your/custom/roles
```

### Calling a Role in a Playbook
``` bash
- name: Configure web server
  hosts: all
  roles:
    - geerlingguy.apache
```
In this example, the geerlingguy.apache role will be applied to all hosts listed in the playbook.

### Using `requirements.yml` to Install Multiple Roles
`requirements.yml` is a file used to define a list of roles and collections to be installed via `ansible-galaxy`. This file helps manage multiple roles in a centralized and consistent manner.
``` bash 
---
- name: geerlingguy.apache
  src: geerlingguy.apache

- name: geerlingguy.mysql
  src: geerlingguy.mysql
```

### Installing Roles from `requirements.yml`
``` bash 
ansible-galaxy install -r requirements.yml
```
This command downloads and installs all the roles listed in the `requirements.yml` file and places them in the default `roles` directory or the directory specified in `ansible.cfg`.

#### Example of a Playbook Using Multiple Roles
``` bash 
---
- name: Configure web and database servers
  hosts: all
  become: true
  roles:
    - geerlingguy.apache
    - geerlingguy.mysql
```

## X. Roles & Rhel-system-roles

### Creating an Apache Role
``` bash
ansible-galaxy --help
```

To create an Ansible role, you use the `ansible-galaxy init` command. This generates a directory structure for the role with basic configuration files.

### Initialize the role
``` bash
cd /path/to/your/custom/roles
ansible-galaxy init apache
```
or you can specify the path
``` bash
ansible-galaxy init --init-path= /path/to/your/custom/roles apache
```
``` bash
cd apache
tree
```

- `handlers/`: Contains handlers, which are tasks triggered by other tasks (for example, to restart a service).
- `tasks/`: Contains the main tasks of the role.
- `templates/`: Contains Jinja2 template files for dynamic file generation.
- `vars/`: Contains variables specific to the role.
### Define Tasks in tasks/main.yml

``` bash
- name: install packages
  yum:
    name: "{{item}}"
    state: present
  loop: "{{packages}}"

- name: Enable services
  service:
    name: "{{item}}"
    state: started
    enabled: yes
  loop: "{{packages}}"
- name: Allow webserver service
  firewalld:
    service: http
    state: enabled
    permanent: yes
    immediate: yes

- name: Create index file from index.html.j2
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
  notify:
    - restart_service
```
Note: `notify` notifies the handler `restart_webservers` when the template file changes.

#### Create the Jinja2 Template in `templates/index.html.j2`
``` bash
Welcome to {{ ansible_fqdn }} on {{ ansible_default_ipv4.address }}
```

### Define the Handler to Restart the Service
``` bash
- name: restart_service
  service:
    name: httpd
    state: restarted
```

### Create the `packages` variable in `vars/main.yml`
``` bash
packages:
- firewalld
- httpd
```

### Create a Playbook Using the Role
``` bash
vim apache.yml
```
``` bash
- name: Install apache from apache-role
  hosts: all
  become: true
  roles:
    - sample-apache
```

### Rhel-system-roles
`rhel-system-roles` is a collection of Ansible roles provided by Red Hat to simplify the management of RHEL (Red Hat Enterprise Linux) systems. Here is a simple guide to install and use these roles, and understand their documentation.

#### Installing rhel-system-roles
``` bash
yum install rhel-system-roles
```
``` bash
sudo cp -r /usr/share/ansible/roles/* /home/ansible/roles1
```

This command downloads and installs the roles in the Ansible roles directory (often ~/.ansible/roles or a directory defined in your configuration). 

#### Documentation of Roles
The documentation for rhel-system-roles roles is typically located in `/usr/share/doc/rhel-system-roles`.
``` bash
ls /usr/share/doc/rhel-system-roles
```

#### TimeSync Role
This role configures the time synchronization service (NTP or chrony).  
Example Playbook using timesync:
``` bash
cd /usr/share/doc/rhel-system-roles/timesync
```
``` bash
cat example-timesync-playbook
```

``` bash
- hosts: all
  vars:
    timesync_ntp_servers:
    - hostname: @server
      iburst: yes
  roles:
  - rhel-system-roles.timesync
```

#### SELinux Role
This role configures SELinux settings.  
Example Playbook using selinux:
``` bash
cat example-multiple-ntp-servers-playbook.yml
```
``` bash
- name: Manage SELinux Booleans
  hosts: all
  become: yes
  vars: 
    selinux_booleans:
    - name: httpd_can_network_connect
      state: true
  roles:
    - rhel-system-roles.selinux
```

Other example:
``` bash
- name: Manage SELinux policy example
  hosts: all
  vars: 
    selinux_policy: targeted
    selinux_state: enforcing
  roles: 
    - rhel-system-roles.selinux
```

## XI. Storage Management
### Partition Creation
The `community.general` collection is required for the LVM modules.
``` bash
ansible-galaxy collection install community.general
```

To create a partition, we use the `parted` module.
``` bash
ansible-doc parted
```

``` bash
---
- name: Create a new partition
  hosts: all
  become: yes
  tasks:
    - name: Create partition
      parted:
        device: /dev/sda
        number: 1
        state: present
        part_end: 1GiB

```

### Format the Partition
Next, we format this newly created partition to ext4 using the `filesystem` module.
``` bash
ansible-doc filesystem
```
``` bash
---
- name: Format the new partition
  hosts: all
  become: yes
  tasks:
    - name: Format partition to ext4
      filesystem:
        fstype: ext4
        dev: /dev/sda1
```

### Mount the Partition and Add it to fstab
``` bash
ansible-doc mount
```
``` bash
- name: Mount the new partition
  hosts: all
  become: yes
  tasks:
    - name: Create mount point directory
      file:
        path: /mnt/data
        state: directory

    - name: Mount partition
      mount:
        path: /mnt/data
        src: /dev/sda1
        fstype: ext4
        state: mounted

```

### Creating a Volume Group with lvg
``` bash
ansible-doc lvg
```
``` bash
---
- name: Create a Volume Group
  hosts: all
  become: yes
  tasks:
    - name: Create volume group
      lvg:
        vg: vg_data  # Nom du groupe de volumes
        pvs: /dev/sdb1  # La partition physique à inclure dans le groupe de volumes
```

### Create a Logical Volume with lvol
``` bash
ansible-doc lvol
```
``` bash
---
- name: Create a Logical Volume
  hosts: all
  become: yes
  tasks:
    - name: Create logical volume
      lvol:
        vg: vg_data  # Le groupe de volumes auquel appartient le LV
        lv: lv_data  # Nom du volume logique
        size: 500M  # Taille du volume logique
```

### Format the Logical Volume with filesystem
``` bash
ansible-doc filesystem
```
``` bash
---
- name: Format the Logical Volume
  hosts: all
  become: yes
  tasks:
    - name: Format logical volume
      filesystem:
        fstype: ext4  # Type de système de fichiers (ex: ext4, xfs)
        dev: /dev/vg_data/lv_data  # Dispositif à formater
```

### Mount the Volume Using the mount Module
``` bash
ansible-doc mount
```
``` bash
---
- name: Mount the Logical Volume
  hosts: all
  become: yes
  tasks:
    - name: Create mount point directory
      file:
        path: /mnt/data1  # Chemin du point de montage
        state: directory  # État souhaité (directory pour créer un répertoire)
    - name: Mount logical volume
      mount:
        path: /mnt/data1  # Chemin du point de montage
        src: /dev/vg_data/lv_data  # Source du dispositif à monter
        fstype: ext4  # Type de système de fichiers
        state: mounted  # État souhaité (mounted pour monter le volume)
```