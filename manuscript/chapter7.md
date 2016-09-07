# Chapter 7: Control Structures
In Chapter 7, we will learn about the aspects of conditionals and iterations that affects program's execution flow in Ansible  
Control structures are of two different type  
* Conditional
* Iterative  

## Conditionals  
Conditionals structures allow Ansible to choose an alternate path. Ansible does this by using *when* statements  
### 7.1 **When** statements  
When statement becomes helpful, when you will want to skip a particular step on a particular host

#### 7.1.1 Selectively calling install tasks based on platform  
  * Edit *roles/apache/tasks/main.yml*,
  ```
  ---
  - include: install.yml
    when: ansible_os_family == 'RedHat'
  - include: start.yml
  - include: config.yml

  ```
  * This will include *install.yml* only if the OS family is Redhat, otherwise it will skip the installation playbook  

#### 7.1.2 Configuring MySQL server based on boolean flag
  * Edit *roles/mysql/tasks/main.yml* and add when statements,  
  ```
  ---
  # tasks file for mysql
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
            bind: "{{ ansible_eth1.ipv4.address }}"

  ```  

#### 7.1.3 Adding conditionals in Jinja2 templates  
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

#### 7.1.4 Running One Time Tasks
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

#### 7.1.5 Conditional Execution of Roles  
  * This will execute app playbook only if the node is running **RedHat** family  
  * Update app.yml to restrict role to be run only on RedHat platform.
  ```

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
  * This will skip app.yml and apache role  
  * Let's change it back to CentOS  
  ```
  ansible-playbook site.yml
  ```  


### 7.2 Iterations  
#### 7.2.1 Iteration over list  
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
#### 7.2.2 Iterating over a Hash Table/Dictionary
  * This iteration can be done with using **with_dict** statement, let us see how  
  * Edit *group_vars/all* file from the **parent directory** and define a dictionary of mysql databases and users to be created

```
---  
#filename: group_vars/all
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