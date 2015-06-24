# ansible-lamp

Provision and configure a LAMP site using ansible on Openstack

Never login to a server again!!


## Setup host_vars and group_vars


## Provision servers webserver1 and database1

~~~
$ ansible-playbook -l localhost provision.yml

PLAY [ensure instance exists in openstack] ************************************ 

GATHERING FACTS *************************************************************** 
ok: [localhost]

TASK: [launch database1 instance] ********************************************* 
changed: [localhost]

TASK: [launch webserver1 instance] ******************************************** 
changed: [localhost]

PLAY RECAP ******************************************************************** 
localhost                  : ok=3    changed=2    unreachable=0    failed=0   
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

## Make sure you can ssh to them using ec2-user and your keys via ansible

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

PLAY [web] ******************************************************************** 

GATHERING FACTS *************************************************************** 
ok: [webserver1]

TASK: [geerlingguy.firewall | Ensure iptables is installed (RedHat).] ********* 
ok: [webserver1]

TASK: [geerlingguy.firewall | Ensure iptables is installed (Dcompanyan).] ********* 
skipping: [webserver1]

TASK: [geerlingguy.firewall | Flush iptables the first time playbook runs.] *** 
ok: [webserver1]

TASK: [geerlingguy.firewall | Copy firewall script into place.] *************** 
ok: [webserver1]
...
...
PLAY RECAP ******************************************************************** 
webserver1                 : ok=43   changed=9    unreachable=0    failed=0   
~~~


~~~
$ ansible-playbook -l database playbooks/db/main.yml 
...
...
~~~


- this next step needs to run after the previous two.
- amoungst other things it adds in */etc/host* entries for both servers 


~~~
$ ansible-playbook -l production playbooks/all/main.yml 

PLAY [all] ******************************************************************** 

GATHERING FACTS *************************************************************** 
ok: [database1]
ok: [webserver1]

TASK: [Build hosts file] ****************************************************** 
changed: [database1] => (item=webserver1)
changed: [webserver1] => (item=webserver1)
changed: [database1] => (item=database1)
skipping: [database1] => (item=localhost)
changed: [webserver1] => (item=database1)
skipping: [webserver1] => (item=localhost)

TASK: [install yumcron] ******************************************************* 
changed: [database1]
changed: [webserver1]

TASK: [start yumcron] ********************************************************* 
changed: [database1]
changed: [webserver1]

TASK: [Ensure admin email is set] ********************************************* 
changed: [database1]
changed: [webserver1]

TASK: [Ensure postfix domain is set] ****************************************** 
changed: [database1] => (item={'line': u'mydomain = mycompany.org.au'})
changed: [webserver1] => (item={'line': u'mydomain = mycompany.org.au'})
changed: [database1] => (item={'line': u'myhostname = mycompany.org.au'})
changed: [webserver1] => (item={'line': u'myhostname = mycompany.org.au'})

NOTIFIED: [new aliases] ******************************************************* 
changed: [database1]
changed: [webserver1]

NOTIFIED: [restart postfix] *************************************************** 
changed: [database1]
changed: [webserver1]

PLAY RECAP ******************************************************************** 
database1                  : ok=8    changed=7    unreachable=0    failed=0   
webserver1                 : ok=8    changed=7    unreachable=0    failed=0   
~~~

- run the following to verify that /etc/hosts is updated on both servers

~~~
$ ansible production -a "cat /etc/hosts"
~~~


## Deploy your site assets

~~~
$ ansible-playbook -l production playbooks/www/deploy-site.yml 

PLAY [web] ******************************************************************** 

GATHERING FACTS *************************************************************** 
ok: [webserver1]

TASK: [check if site exists] ************************************************** 
ok: [webserver1]

TASK: [the site exists] ******************************************************* 
skipping: [webserver1]

TASK: [copy the web application] ********************************************** 
changed: [webserver1]

PLAY RECAP ******************************************************************** 
webserver1                 : ok=3    changed=1    unreachable=0    failed=0   

~~~

- similarly for your database you can import a mysql dump

~~~
$ ansible-playbook -l production playbooks/db/deploy-site.yml 

PLAY [database] *************************************************************** 

GATHERING FACTS *************************************************************** 
ok: [database1]

TASK: [Import database] ******************************************************* 
ok: [database1] => {
    "msg": "Database is drupal_company"
}

TASK: [Copy database dump to remote host and import] ************************** 
ok: [database1]

TASK: [Import database dump] ************************************************** 
changed: [database1]

PLAY RECAP ******************************************************************** 
database1                  : ok=4    changed=1    unreachable=0    failed=0   
~~~

