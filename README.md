# ansible-lamp

Provision and configure a LAMP site using ansible on Openstack

This assumes you have ansible setup on your devops workstation

see [How To Setup Desktop For Openstack/Ansible](HOWTO_ANSIBLE.md)


### Setup you project

- assumes you have setup a Python virtualenv. See previous Howto link.
~~~
$ workon website
$ git clone https://github.com/dannysheehan/ansible-lamp .
~~~

- source role requirements from Ansible Galaxy
~~~
$ ansible-galaxy install -r requirements.txt
~~~


### Generate OpenStack keys to access nodes as ec2-user

- Ansible will use public keys and sudo to root to configure your sites

~~~
$ ssh-keygen -y -f website > website.pub

$ nova keypair-add --pub_key website.pub website

$ nova keypair-list
+----------+-------------------------------------------------+
| Name     | Fingerprint                                     |
+----------+-------------------------------------------------+
| website  | 12:e4:1b:xx:6d:xx:ee:e0:54:5d:fa:7c:d1:xx:xx:f5 |
+----------+-------------------------------------------------+

$ mkdir -p ../private
$ cp website ../private/website.pem
$ chmod 600 ../private/website.pem
~~~


### Setup your Inventory file

I like to keep this under *../private/production*.


**Example:**

NOTE: You won't know your ansible_ssh_host names until your provision them in OpenStack.
You will come back later and fill them in (marked TBD below)

~~~
[localhost]
localhost

[localhost:vars]
ansible_connection=local
ansible_python_interpreter="/usr/bin/env python"

[database]
database1  ansible_ssh_host=TBD  mysql_replication_role=master

[web]
webserver1 ansible_ssh_host=TBD

[production:children]
web
database

[production:vars]
ansible_ssh_user=ec2-user
ansible_ssh_private_key_file=../private/website.pem
#
# galaxy role - geerlingguy.repo-epel bug workaround
ansible_distribution_major_version="6"
~~~

### Setup host_vars and group_vars

- this is an example of the types of things you can configure.
- look at the playbooks and roles for more details.

**host_vars/localhost**

- your OpenStack instance flavor, availability zones and security groups you want to use etc. Use `nova list` commands to find what is avalable in OpenStack.

~~~
$ more host_vars/localhost
---

os_security_groups: "PublicNodes"
os_image_id: "a5e74703-f343-415a-aa23-bd0f0aacfc9e"
os_availability_zone: "melbourne"
os_flavor_id: "639b8b2a-a5a6-4aa2-8592-ca765ee7af63"
os_key_name: "website"

~~~

**group_vars/**

- your database name, passwords, vhost configurations for your website etc.

~~~
 more group_vars/*


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

- this will create your nodes in OpenStack.

~~~
$ ansible-playbook -l localhost provision.yml
~~~

- get the ip addresses of the two nodes you just created.

~~~
$ nova list | egrep "database1|webserver1"
| 36169c8b-0375-4133-960e-196e53333333 | database1   | ACTIVE  | -   | Running     | berry=115.10.10.191  |
| 4039e066-0e69-4291-b455-733333908cf1 | webserver1  | ACTIVE  | -   | Running     | berry=115.10.10.192  |
~~~

- plug them into the inventory file your created previously *production* (the TBDs)

### Make you can connect to your nodes via ansible

- if you setup your *website.pem* file correctly in your *production* inventory file for ansible to find this should work.

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
- later on you can do a `ansible-playbook -l production configure.yml` if you want to do it all in one.

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
