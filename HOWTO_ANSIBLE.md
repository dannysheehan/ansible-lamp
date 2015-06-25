# How To Setup Your Workstation For OpenStack/Ansible Devops

The following assumes you have a Ubuntu desktop.

### Setup python virtual environment

- install pip if you do not have it already
- [pip documentation](https://pip.pypa.io/en/latest/installing.html)

~~~
# Make sure you have > 2.7 of python installed.
$ python --version

$ wget https://bootstrap.pypa.io/get-pip.py

$ python get-pip.py

$ aptitiude install python-devel

# You can upgrade later
$ pip install -U pip
~~~

- install virtualenv

~~~
$ pip install virtualenv
$ pip install virtualenvwrapper
~~~

- setup where you want to keep your projects
- put this in your *.bashrc*

~~~
# Location of virtual environments.
export WORKON_HOME=$HOME/.virtualenvs
export PROJECT_HOME=$HOME/
source /usr/bin/virtualenvwrapper.sh
~~~

### Create your OpenStack project

- mkproject and workon allow you to setup and work on a separate python module environment

~~~
$ mkproject websitex
$ workon websitex
~~~


### Install Openstack clients

~~~
$ pip install requests[security]
$ pip install python-novaclient
$ pip install python-glanceclient
$ pip install python-heatclient
~~~


### Source your openstack project

- you need to download Project openrc.sh file from your OpenStack dashboard.
- [instructions](http://docs.openstack.org/cli-reference/content/cli_openrc.html)

- comment out the lines requesting a password  `<project>-openrc.sh`
- obtain your password from OpenStack <username> -> Settings -> Reset Password

~~~
#echo "Please enter your OpenStack Password: "
#read -sr OS_PASSWORD_INPUT
export OS_PASSWORD="xxxxx"

$ source ./<project>-openrc.sh
~~~

### Test your OpenStack connection

~~~
$ nova availability-zone-list
+----------------+-----------+
| Name           | Status    |
+----------------+-----------+
| monash         | available |
| QRIScloud      | available |
...
...
~~~

### Install ansible on your desktop

~~~
$ pip install ansible
~~~
