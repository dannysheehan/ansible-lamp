# ansible-lamp

Provision and configure a LAMP site using ansible on Openstack

This assumes you have ansible setup on your devops workstation


### Setup host_vars and group_vars

- this is an example of the types of things you can configure.
- look at the playbooks and roles for more details.
~~~
 more group_vars/*
::::::::::::::
group_vars/database
::::::::::::::
---

::::::::::::::
group_vars/production
::::::::::::::
---
server_admin: "admin@mail"
server_domain: "company.org.au"

mysql_users:
  - name: drupaluser
    host: "%"
    password: mypassword
    priv: "*.*:ALL"

mysql_databases:
  - name: mydatabase
    collation: utf8_general_ci
    encoding: utf8

nova_compute:
  - key_name: mycompany
  - availability_zone: "melbourne"
::::::::::::::
group_vars/web
::::::::::::::

php_date_timezone: "Australia/Melbourne"

apache_create_vhosts: true
apache_vhosts_filename: "vhosts.conf"
apache_vhosts:
  - {
    servername: ""
    serveralias: ""
...
...
~~~

### Provision servers webserver1 and database1

~~~
$ ansible-playbook -l localhost provision.yml
~~~

- get the ip's
~~~
$ nova list | egrep "database1|webserver1"
| 36169c8b-0375-4133-960e-196e53333333 | database1   | ACTIVE  | -   | Running     | berry=115.10.10.191  |
| 4039e066-0e69-4291-b455-733333908cf1 | webserver1  | ACTIVE  | -   | Running     | berry=115.10.10.192  |
~~~

- plug them into your inventory file *production*
- this is a symbolick link to *../priviate/production*
~~~
[localhost:vars]
ansible_connection=local
ansible_python_interpreter=~/.virtualenvs/heat/bin/python

[database]
database1  ansible_ssh_host=115.10.10.191 mysql_replication_role=master

[web]
webserver1 ansible_ssh_host=115.10.10.192

[production:children]
web
database

[production:vars]
ansible_ssh_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/production.pem
ansible_distribution_major_version="6"
~~~

### Make sure you can ssh to them using ec2-user and your keys via ansible

~~~
$ ansible webserver1 -i production -a "hostname"
webserver1 | success | rc=0 >>
webserver1

$ ansible database1 -i production -a "hostname"
database1 | success | rc=0 >>
database1
~~~


### Configure webserver1 and database1 with your playbooks 

- do one playbook at a time first time around.
~~~
$ ansible-playbook -l web playbooks/www/main.yml 
~~~

~~~
$ ansible-playbook -l database playbooks/db/main.yml 
~~~

- this next step needs to run after the previous two.
- among other things it adds in */etc/host* entries for both servers 


~~~
$ ansible-playbook -l production playbooks/all/main.yml 
~~~

- run the following to verify that /etc/hosts is updated on both servers

~~~
$ ansible production -a "cat /etc/hosts"
~~~


## Deploy your site assets

- this deploys your */var/www/html* assets
~~~
$ ansible-playbook -l production playbooks/www/deploy-site.yml 
~~~

- similarly for your database you can import a mysql dump
~~~
$ ansible-playbook -l production playbooks/db/deploy-site.yml 
~~~

