# Control Structures

In Chapter 7, we will learn about the aspects of conditionals and iterations that affects program's execution flow in Ansible  
Control structures are of two different type

* Conditional  
* Iterative  

## Conditionals

Conditionals structures allow Ansible to choose an alternate path. Ansible does this by using *when* statements

## **When** statements

When statement becomes helpful, when you will want to skip a particular step on a particular host

### Selectively calling install tasks based on platform

* Edit *roles/apache/tasks/main.yml*,

```
---
- include: install.yml
  when: ansible_os_family == 'RedHat'
- include: start.yml
- include: config.yml
```

* This will include *install.yml* only if the OS family is Redhat, otherwise it will skip the installation playbook

### Configuring MySQL server based on boolean flag

* Edit *roles/mysql/tasks/main.yml* and add when statements,

```
---
- include: install.yml

- include: start.yml
  when: mysql.server

- include: config.yml
  when: mysql.server
```

* Edit *db.yml* as follows,

```
---
- name: Playbook to configure DB Servers
  hosts: db
  become: true
  roles:
  - mysql
  vars:
    mysql:
      server: true
      config:
        bind: "{{ ansible_eth0.ipv4.address }}"
```


### Adding conditionals in Jinja2 templates

* Put the following content in *roles/mysql/templates/my.cnf.j2*

```
[mysqld]

{% if mysql.config.datadir is defined %}
datadir={{ mysql['config']['datadir'] }}
{% endif %}

{% if mysql.config.socket is defined %}
socket={{ mysql['config']['socket'] }}
{% endif %}

symbolic-links=0
log-error=/var/log/mysqld.log

{% if mysql.config.pid is defined %}
pid-file={{ mysql['config']['pid']}}
{% endif %}

[client]
user=root
password={{ mysql_root_db_pass }}
```

* These conditions will run flawlessly, because we have already defined these Variables

### Running One Time Tasks

* To see how this works, lets take a look at the code in *roles/mysql/tasks/config.yml*

```
    [...]
- name: reset default root password
shell: mysql --user=root --password="{{ MYSQL_DEFAULT_PASS }}" --connect-expired-password mysql < /root/.mysql_reset_pass.sql
run_once: true
ignore_errors: yes
     [...]
```

* In some cases there may be a need to only run a task one time and only on one host. This can be achieved by configuring “run_once” on a task

### Conditional Execution of Roles

* This will execute app playbook only if the node is running **RedHat** family
* Update app.yml to restrict role to be run only on RedHat platform.

```
---
  - name: Playbook to configure App Servers
    hosts: app
    become: true
    vars:
      fav:
        fruit: mango
    roles:
    - { role: apache, when: ansible_os_family == 'RedHat' }
```

* Let's run this code

```
ansible-playbook site.yml
```
[Output]

```
TASK [setup] *******************************************************************
ok: [192.168.61.12]
ok: [192.168.61.13]

TASK [base : create admin user] ************************************************
skipping: [192.168.61.12]
skipping: [192.168.61.13]

TASK [base : remove dojo] ******************************************************
skipping: [192.168.61.12]
skipping: [192.168.61.13]

TASK [base : install tree] *****************************************************
skipping: [192.168.61.12]
skipping: [192.168.61.13]

TASK [base : install ntp] ******************************************************
skipping: [192.168.61.12]
skipping: [192.168.61.13]

TASK [base : start ntp service] ************************************************
skipping: [192.168.61.12]
skipping: [192.168.61.13]

TASK [apache : Installing Apache...] *******************************************
skipping: [192.168.61.12]
skipping: [192.168.61.13]

TASK [apache : Starting Apache...] *********************************************
skipping: [192.168.61.12]
skipping: [192.168.61.13]

TASK [apache : Creating configuration from templates...] ***********************
skipping: [192.168.61.12]
skipping: [192.168.61.13]

TASK [apache : Copying index.html file...] *************************************
skipping: [192.168.61.12]
skipping: [192.168.61.13]

```

**Exercise**: Try using **Debian** instead of **RedHat** . You shall see app role being skipped altogether. Don't forget to put it back after you try this out.

## Iterations

### Iteration over list

* Create a list of packages  
* Let us create the following list of packages in base role.  
* Edit *roles/base/defaults/main.yml* and put

```
---
# packages list
demolist:
  packages:
    - atk
    - flac
    - eggdbus
    - pixman
    - polkit

```

* Also edit *roles/base/tasks/main.yml* to iterate over this list of items and install packages

```
- name: install a list of packages
  yum:
    name: "{{ item }}"
  with_items: {{ demolist.packages }}
```

* Let's check the output

```
TASK [base : install a list of packages] ***************************************
changed: [192.168.61.12] => (item=[u'atk', u'flac', u'eggdbus', u'polkit', u'pixman'])
changed: [192.168.61.13] => (item=[u'atk', u'flac', u'eggdbus', u'polkit', u'pixman'])
```

### Iterating over a Hash Table/Dictionary

* This iteration can be done with using **with_dict** statement, let us see how.
* Edit *group_vars/all* file from the **parent directory** and define a dictionary of mysql databases and users to be created

```
---
  fav:
    color: blue
    fruit: peach
  mysql_bind: "{{ ansible_eth0.ipv4.address }}"
  mysql:
    databases:
      infinity:
        state: present
      peace:
        state: present
    users:
      dojo:
        pass: PassWord@1234
        host: '%'
        priv: '*.*:ALL'
        state: present
      koko:
        pass: f8Usg3ord@1we28
        host: '%'
        priv: '*.*:ALL'
        state: present

```

* **Append** the following iteration in *roles/mysql/tasks/config.yml*

```
- name: create mysql databases
  mysql_db:
    name: "{{ item.key }}"
    state: "{{ item.value.state }}"
  with_dict: "{{ mysql['databases'] }}"

- name: create mysql users
  mysql_user:
    name: "{{ item.key }}"
    host: "{{ item.value.host }}"
    password: "{{ item.value.pass }}"
    priv: "{{ item.value.priv }}"
    state: "{{ item.value.state }}"
  with_dict: "{{ mysql['users'] }}"
```

* Execute the *db* playbook to verify the output

```
ansible-playbook db.yml
```

## Exercises

* Define dictionary of properties for a new database user  in group_vars/all. Observe if it gets created automatically  output by running db.yml playbook. Validate if the user is been actually present by logging on to the mysql server and checking status.
* Update index.html.j2 to iterate over the dictionary of favorites and generate html content to display it instead of adding multiple lines.
* Define a hash/dictionary  of apache virtual hosts to be created  and create a template which would iterate over that dictionary and create vhost configurations.
* Learn about what else you could loop over, as well as how to do so by reading this document http://docs.ansible.com/ansible/playbooks_loops.html#id12
