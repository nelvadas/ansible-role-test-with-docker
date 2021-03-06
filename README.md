Testing Ansible Roles with docker containers
--------------------------------------------





The purpose of this tutorial is to show you how you can leverage docker to Automagically provision container on the fly
to test an ansible playbooks and roles.
Various initiatives have been started around Ansible tests.

Let's consider the following *site.yml* playbook with a single *hellworld* role.
The helloworld role perform one basic action: touch a new file /tmp/hello.txt on the host
```
---

- hosts: localhost
  connection: local
  roles:
    - helloworld

```

*Let's run the playbook*

```
$ ansible-playbook site.yml

PLAY [localhost] ***************************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************ok: [localhost]

TASK [helloworld : Create /tmp/hello.txt] ************************************************************************************************changed: [localhost]

PLAY RECAP *********************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0
```
the task responsible of the file creation is hosted in the helloword/tasks/main.yml

```
  1 ---
  2 # tasks file for roles/helloworld
  3
  4 - name: "Create {{ tmpFile}}"
  5   file:
  6       path: "{{ tmpFile }}"
  7       state: touch
```

*How can we test the following role using ansible and Docker?*

# docker_provision role

Chris Meyers provides a [docker_provision](https://github.com/chrismeyersfsu/provision_docker/) role to provision a docker container for an inventory machine 
pull the role in your project's role using ansible galaxy or a traditionnal git clone command

```
$ cd roles
git clone https://github.com/chrismeyersfsu/provision_docker/

```
In the next sections, we will rely on this docker_provision role to automatically populate docker containers and run the helloword on them as
you run the playbook earlier on your localhost. 


# Create a test folder structrue
Each role created with galaxy has a default *tests* folder.
create a test.yml file inside the test folder.
we are going to leverage the roles/helloworld/tests/test.yml to perform a basic Ansible test.
## Reference the role to be tested and its dependencies
The test.yml playbook is intended to test the helloworld role, so in the role subfolder, 
instead of recreating the entire helloword structure, we are going to create a symbolic link to the existing one
Do the same for the dependent docker_provision role
```
$ ln -s ../../../helloworld helloworld
$ ln -s ../../../provision_docker provision_docker
```
Verify the tests subfolder structure 

```
$cd roles/helloworld/tests
$ tree
.
├── Dockerfile
├── inventory
├── roles
│   ├── helloworld -> ../../../helloworld
│   ├── provision_docker -> ../../../provision_docker
└── test.yml

3 directories, 4 files

```
## Provide a test inventory

* The Dockerfile contains a set of instruction used to prepared containers images ( not mandatory) you can relies on your own images
```
  4 FROM alpine
  5
  6 RUN apk add --update \
  7     python \
  8     python-dev \
  9     py-pip \
 10     build-base \
 11   && pip install virtualenv \
 12   && rm -rf /var/cache/apk/*
```
From a regular alpine, we some python utilities, to enable ansible fact gathering for example.
Then build a mix called alpy:= alpine + python :)
```
$ docker build -t alpy:latest .
```


* The inventory file will be used to pass the inventory variables when running the test.yml playbook
```
   # inventory
   [helloserver]
   cnt001 image="alpy:latest"
```
in our inventory we declare on host in the helloserver category: a docker container with name cnt001 will be created to mock this role using the prebuilt
alpy:latest image.

## Assemble the blocks in a test playbook

* test.yml contains various instructions to : Provision docker containers based on inventory declration, run the playbook on thoses container and verify assertions

```
  1 ---
  2 - name: Bring up docker containers
  3   hosts: localhost
  4   gather_facts: false
  5   roles:
  6     - role: provision_docker
  7       provision_docker_inventory_group: "{{ groups['helloserver'] }}"
  8
  9
 10 - name: Play HelloWorld Role
 11   hosts: helloserver
 12   roles:
 13     - helloworld
 14
 15 - name: Verify Tests
 16   hosts: helloserver
 17   vars:
 18     - tmpFile: "/tmp/file.txt"
 19   tasks:
 20     - name: "Assert {{ tmpFile }} exists"
 21       stat:
 22         path: "{{ tmpFile }}"
 23       register: assertFileExist
 24       failed_when: assertFileExist.stat.exists == false 

```




### Provision docker containers
The provision_docker role is invoked to create one docker container for each host listed in helloservergroup

### Trigger the helloworld role tasks
After instanciating docker container, we can run the helloworld tasks on helloword docker containers.

### Check some assertions.
After the playbook execution, we can perform some checks on the installed containers.
here for example we ensure the hello.txt file exists.


## Run the playbook

```
$ ansible-playbook -i inventory test.yml

PLAY [Bring up docker containers] ****************************************************************************************************************************************

TASK [provision_docker : Ensure required vars are defined] ***************************************************************************************************************
skipping: [localhost]

TASK [provision_docker : Gather facts] ***********************************************************************************************************************************
ok: [localhost]

TASK [provision_docker : Install libselinux-python package for 'copy' task] **********************************************************************************************
skipping: [localhost]

TASK [provision_docker : Bring up inventory group of hosts] **************************************************************************************************************
changed: [localhost] => (item=cnt001)

TASK [provision_docker : Get IP of container] ****************************************************************************************************************************
ok: [localhost -> localhost] => (item=cnt001)

TASK [provision_docker : Associate ip address with hosts] ****************************************************************************************************************
skipping: [localhost] => (item=(0, 'cnt001'))

TASK [provision_docker : Wait for ssh] ***********************************************************************************************************************************
skipping: [localhost] => (item=cnt001)

TASK [provision_docker : Make sure ssh is really up] *********************************************************************************************************************
skipping: [localhost] => (item=cnt001)

TASK [provision_docker : Add docker hosts with connection docker] ********************************************************************************************************
ok: [localhost] => (item=(0, 'cnt001'))

TASK [provision_docker : Bring up list of hosts] *************************************************************************************************************************

TASK [provision_docker : Get IP of container] ****************************************************************************************************************************

TASK [provision_docker : Associate ip address with hosts] ****************************************************************************************************************

TASK [provision_docker : Wait for ssh] ***********************************************************************************************************************************

TASK [provision_docker : Add docker hosts with connection docker] ********************************************************************************************************

TASK [provision_docker : Make sure able to connect to hosts] *************************************************************************************************************

PLAY [Play HelloWorld Role] **********************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [cnt001]

TASK [helloworld : Create /tmp/hello.txt] ********************************************************************************************************************************
changed: [cnt001]

PLAY [Verify Tests] ******************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [cnt001]

TASK [Assert /tmp/hello.txt exists] **************************************************************************************************************************************
ok: [cnt001]

PLAY RECAP ***************************************************************************************************************************************************************
cnt001                     : ok=4    changed=1    unreachable=0    failed=0
localhost                  : ok=4    changed=1    unreachable=0    failed=0

```





 


