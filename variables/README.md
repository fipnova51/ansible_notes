# Notes on ansible variables

The syntax to define a variable depends on where you define it

in an invenory file | in a playbook or vars file
---|---
foo=bar | foo: bar

Ansible provides a set of `ansible variables` that you can use within your playbbok.
This list is available through `ansible -m setup` output

## Playbook variables
The syntax to call/use a variable is `{{ variable_name }}`.There are different ways
to use variables in a playbook

let's use the [playbook below](vars_01.yml) containing a variable `target`

```python
---
- hosts: '{{ target }}'
  tasks:
     - name: display value of variable target
       debug: msg="The variable target has the value  {{ target }}"
...
```

#### pass the value in command line
the value of `target` can be provided when calling the playbook
```shell
vagrant@ansible-master:/vagrant/variables$ ansible-playbook /tmp/playbook.yml --extra-vars "target=192.168.33.12"  
PLAY [192.168.33.12] ***********************************************************
TASK [Gathering Facts]**********************************************************
ok: [192.168.33.12]
TASK [display value of variable target] ****************************************
ok: [192.168.33.12] => {                                                                                           
    "msg": "The variable target is set to 192.168.33.12"                                                           
}
PLAY RECAP *********************************************************************
192.168.33.12              : ok=2    changed=0    unreachable=0    failed=0
```

#### variables can be defined in the vars section of a Playbook
Variable `foo` is defined in the `vars` section. See [playbook below](vars_02.yml)
```python
---
- hosts: '{{ target }}'
  vars:
    foo: bar
  tasks:
    - name: display value of foo
      debug: msg="The variable foo is set to {{ foo }}"
...

```
Result:
```shell
vagrant@ansible-master:~$ ansible-playbook /tmp/playbook.yml --extra-vars "target=192.168.33.12"

PLAY [192.168.33.12] ***********************************************************
TASK [Gathering Facts] *********************************************************
ok: [192.168.33.12]

TASK [display value of foo] ****************************************************
ok: [192.168.33.12] => {
    "msg": "The variable foo is set to bar"
}

PLAY RECAP *********************************************************************
192.168.33.12              : ok=2    changed=0    unreachable=0    failed=0
```

#### variables can be defined in the vars_files section
`vars_files` section contain a reference to a file containing the variables definition.
This file must be in the same folder as the playbook.
This [playbook](vars_03.yml) is referencing file `vars.yml` containing the value of `foo`
```python
---
- hosts: '{{ target }}'
  vars_files:
    - vars.yml
  tasks:
    - name: display value of foo
      debug: msg="The variable foo is set to {{ foo }}"
...
```
```shell
vagrant@ansible-master:~$ cat /tmp/vars.yml
foo: bar
```
Result:
```shell
vagrant@ansible-master:~$ ansible-playbook /tmp/playbook.yml --extra-vars "target=192.168.33.12"

PLAY [192.168.33.12] ***********************************************************

TASK [Gathering Facts]**********************************************************
ok: [192.168.33.12]

TASK [display value of foo] ****************************************************
ok: [192.168.33.12] => {
    "msg": "The variable foo is set to bar"
}

PLAY RECAP *********************************************************************
192.168.33.12              : ok=2    changed=0    unreachable=0    failed=0
```

## Inventory variables
Variables can be defined in an `inventory` file but this not recommended because
adding several variables can make the file overloaded.

Example of [inventory file](vars_inventory_01.yml). Server *12* and *13* has a specific variable while a default variable is assigned to the group *app* (the code is set
  in default invetory file /etc/ansible/hosts)
```python
# Application servers
[app]
192.168.33.12 var1='value of var1 specific to 192.168.33.12'
192.168.33.13 var1='value of var1 specific to 192.168.33.13'

[app:vars]
var_app_group='value for all servers in app group'
```
Example of [playbook](vars_04.yml)
```python
---
- hosts: '{{ target }}'

  tasks:
  - name: display variable assigned to group app
    debug: msg='{{ var_app_group }}'

  - name: display variable per server
    debug: msg='{{ var1 }}'
...
```
Result:
```shell
vagrant@ansible-master:~$ ansible-playbook ~/playbook.yml --extra-vars "target=app"

PLAY [app]**********************************************************************

TASK [Gathering Facts]**********************************************************
ok: [192.168.33.12]
ok: [192.168.33.13]

TASK [display variable assigned to group app]***********************************
ok: [192.168.33.12] => {
    "msg": "value for all servers in app group"
}
ok: [192.168.33.13] => {
    "msg": "value for all servers in app group"
}

TASK [display variable per server]**********************************************
ok: [192.168.33.12] => {
    "msg": "value of var1 specific to 192.168.33.12"
}
ok: [192.168.33.13] => {
    "msg": "value of var1 specific to 192.168.33.13"
}

PLAY RECAP**********************************************************************
192.168.33.12              : ok=3    changed=0    unreachable=0    failed=0
192.168.33.13              : ok=3    changed=0    unreachable=0    failed=0
```

The alternative is to define those variables in `host_vars` / `group_vars` files.
The syntax is
* `hostname` file defined under `etc/ansible/host_vars/<hostname>` or `<playbook_dir>
/host_vars/<hostname>`
* `groupname` file defined under `etc/ansible/group_vars/<groupname>` or `<playbook_dir>
/group_vars/<groupname>`

Example, my `inventory` file doesn't contain any variables
```shell
vagrant@ansible-master:~$ egrep "^#|^$" -v /etc/ansible/hosts
[app]
192.168.33.12
192.168.33.13
```
they're moved to their corresponding `host_vars` and `group_vars` vars files.
```shell
vagrant@ansible-master:~$ find /etc/ansible/ -ls
262426    4 drwxr-xr-x   5 root     root         4096 May  8 12:13 /etc/ansible/
267912    4 drwxr-xr-x   2 root     root         4096 May  8 12:15 /etc/ansible/host_vars
268050    4 -rw-r--r--   1 root     root           48 May  8 12:15 /etc/ansible/host_vars/192.168.33.12
268051    4 -rw-r--r--   1 root     root           48 May  8 12:15 /etc/ansible/host_vars/192.168.33.13
268049    4 drwxr-xr-x   2 root     root         4096 May  8 12:16 /etc/ansible/group_vars
268052    4 -rw-r--r--   1 root     root           52 May  8 12:16 /etc/ansible/group_vars/app
268054    4 -rw-r--r--   1 root     root         1420 May  8 12:13 /etc/ansible/hosts
```
Result:
```shell
vagrant@ansible-master:~$ ansible-playbook ~/playbook.yml --extra-vars "target=app"

PLAY [app]**********************************************************************

TASK [Gathering Facts]**************************************************************************
ok: [192.168.33.12]
ok: [192.168.33.13]

TASK [display variable assigned to group app]***********************************
ok: [192.168.33.12] => {
    "msg": "value for all servers in app group"
}
ok: [192.168.33.13] => {
    "msg": "value for all servers in app group"
}

TASK [display variable per server]**********************************************
ok: [192.168.33.12] => {
    "msg": "value of var1 specific to 192.168.33.12"
}
ok: [192.168.33.13] => {
    "msg": "value of var1 specific to 192.168.33.13"
}

PLAY RECAP**********************************************************************
192.168.33.12              : ok=3    changed=0    unreachable=0    failed=
```

Same result is achieved when those files are in the *playbook directory*
```shell
vagrant@ansible-master:~$ find /etc/ansible/ -ls
262426    4 drwxr-xr-x   5 root     root         4096 May  8 12:13 /etc/ansible/
267912    4 drwxr-xr-x   2 root     root         4096 May  8 12:23 /etc/ansible/host_vars
268049    4 drwxr-xr-x   2 root     root         4096 May  8 12:23 /etc/ansible/group_vars
268054    4 -rw-r--r--   1 root     root         1420 May  8 12:13 /etc/ansible/hosts

vagrant@ansible-master:~$ find ~ -ls
131077    4 drwxr-xr-x   7 vagrant  vagrant      4096 May  8 12:22 /home/vagrant
268048    4 drwxrwxr-x   2 vagrant  vagrant      4096 May  8 12:23 /home/vagrant/host_vars
268055    4 -rw-r--r--   1 vagrant  vagrant        48 May  8 12:23 /home/vagrant/host_vars/192.168.33.12
268056    4 -rw-r--r--   1 vagrant  vagrant        48 May  8 12:23 /home/vagrant/host_vars/192.168.33.13
268053    4 drwxrwxr-x   2 vagrant  vagrant      4096 May  8 12:23 /home/vagrant/group_vars
268057    4 -rw-r--r--   1 vagrant  vagrant        52 May  8 12:23 /home/vagrant/group_vars/app
```
Result:
```shell
vagrant@ansible-master:~$ ansible-playbook ~/playbook.yml --extra-vars "target=app"

PLAY [app]**********************************************************************

TASK [Gathering Facts]**********************************************************
ok: [192.168.33.13]
ok: [192.168.33.12]

TASK [display variable assigned to group app]***********************************
ok: [192.168.33.12] => {
    "msg": "value for all servers in app group"
}
ok: [192.168.33.13] => {
    "msg": "value for all servers in app group"
}

TASK [display variable per server]*************************************************************************
ok: [192.168.33.12] => {
    "msg": "value of var1 specific to 192.168.33.12"
}
ok: [192.168.33.13] => {
    "msg": "value of var1 specific to 192.168.33.13"
}

PLAY RECAP**********************************************************************
192.168.33.12              : ok=3    changed=0    unreachable=0    failed=0
192.168.33.13              : ok=3    changed=0    unreachable=0    failed=0
```
## Magic variables, accessing a variable defined for a different host
It's the ability to use a variable defined for another host, everything is explained
[here](http://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#magic-variables-and-how-to-access-information-about-other-hosts)

Example: the [playbook](vars_05.yml) will print the value assigned to host 192.168.33.12
```python
---
- hosts: '{{ target }}'

  tasks:
  - name: display variable assigned to group app
    debug: msg='{{ var_app_group }}'

  - name: display variable per server
    debug: msg='{{ hostvars['192.168.33.12']['var1'] }}'
...
```
Result: the playbook targets two differents hosts but display the same output
```shell
vagrant@ansible-master:~$ ansible-playbook ~/playbook.yml --extra-vars "target=app"

PLAY [app]**********************************************************************

TASK [Gathering Facts]**********************************************************
ok: [192.168.33.12]
ok: [192.168.33.13]

TASK [display variable assigned to group app]***********************************
ok: [192.168.33.12] => {
    "msg": "value for all servers in app group"
}
ok: [192.168.33.13] => {
    "msg": "value for all servers in app group"
}

TASK [display variable per server]**********************************************
ok: [192.168.33.12] => {
    "msg": "value of var1 specific to 192.168.33.12"
}
ok: [192.168.33.13] => {
    "msg": "value of var1 specific to 192.168.33.12"
}

PLAY RECAP**********************************************************************
192.168.33.12              : ok=3    changed=0    unreachable=0    failed=
```

## Facts, variables derived from the system
Facts (variables derived from the system running the tasks) are gathered everytime
a playbook is executed (this can be disabled with `gather_facts: no`). The list of
variables can be retrieved with `ansible -m setup`

Because you can have variables nested in other variables, you can access a value
with the notation `[var1][var1_.1]` or `var1.var1_1`

Example of [playbook](vars_06.yml)
```python
---
- hosts: '{{ target }}'

  tasks:
  - name: display variable per server
    debug: msg='{{ ansible_default_ipv4.macaddress }}'
...
```
Result:
```shell
vagrant@ansible-master:~$ ansible-playbook ~/playbook.yml --extra-vars "target=app"

PLAY [app]**********************************************************************

TASK [Gathering Facts]**************************************************************************
ok: [192.168.33.12]
ok: [192.168.33.13]

TASK [display variable per server]**********************************************
ok: [192.168.33.12] => {
    "msg": "08:00:27:97:35:7d"
}
ok: [192.168.33.13] => {
    "msg": "08:00:27:97:35:7d"
}

PLAY RECAP**********************************************************************
192.168.33.12              : ok=2    changed=0    unreachable=0    failed=0
192.168.33.13              : ok=2    changed=0    unreachable=0    failed=0
```

## Environment variables
It's possible to get a value of an environment variable, store it in a variable
then display the value.

Example of [playbook](vars_07.yml)
```python
---
- hosts: '{{ target }}'

  tasks:
     - name: define variable in .profile file
       lineinfile: dest=~/.profile regexp=^SIMON line=SIMON='hello ansible'

     - name: get the value of variable SIMON in .profile file
       shell: '. ~/.profile && echo $SIMON'
       register: foo

     - name: display value of variable SIMON
       debug: msg="The variable SIMON is set to {{ foo.stdout }}"
...
```
* `hosts` is defined through a variable provided on command line, depending the
value it can target *all* servers / a *group* of servers defined in your inventory
file or *individual* server
* first task creates / replaces a variable `SIMON` defined in file `~/.profile`
* second task executes a `shell` command to display the value of `SIMON` and `register`
the content in a variable named `foo`
* last task display the content of `foo`.
