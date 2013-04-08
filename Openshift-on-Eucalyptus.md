## Introduction

This article explains how to setup openshift origin on a, Fedora 18 instance running on a Eucalyptus private cloud. In order to get Fedora 18 up and running as an instance on Eucalyptus one should follow this article , [Fedora 18 EMI](https://github.com/eucalyptus/eucalyptus/wiki/Fedora-18-Image)

## Setup

The setup is very simple (POC based), it consists of 1 instance running Fedora 18, this instance would act as both broker as well as node in the openshift setup.

The instance type chosen is m1.xlarge that provides 2CPU/2048MB RAM/20GB of disk, the EMI is bfEBS hence providing persistent disk for root filesystem.

## Installation and Configuration

In order to setup Openshift origin we rely on the [openshift ansible playbook](https://github.com/maxamillion/ansible-openshift_origin)

### Install git and ansible 

On the local machine get ansible and git installed:

```
yum -y install git ansible
```

### Configure ansible hosts

Edit the ansible hosts file (/etc/ansible/hosts) on to add broker and node information

```
[brokers]
10.104.3.10

[nodes]
10.104.3.10
```

Note that 10.104.3.10 is public IP address of the instance, replace with correct settings.

### Get the ansible playbook

```
git clone https://github.com/maxamillion/ansible-openshift_origin.git
```

Currently there is a problem, the playbook tries to install gcc that tries to update glibc on the instance, that in turns fails complaining for audit package been old version.

```
Transaction Check Error:
file /usr/lib64/audit from install of glibc-2.16-30.fc18.x86_64 conflicts with file from package audit-2.2.1-2.fc18.x86_64
```

It is hence advice to modify the broker.yml to include audit in the develpkgs task, see below

```
- name: Devel Packages Install
  hosts: brokers
  user: root
  vars_files:
    - varfiles/origin_vars.yml
  tasks:
    ### Dev tools needed by some gem installs for native modules/extensions
    - name: Install dev-tool deps for gem installs
      yum: pkg=$item state=latest
      with_items:
        - ruby-devel
        - audit
        - mysql-devel
        - mongodb-devel
        - gcc
        - make
  tags:
    - develpkgs
```

### Run the ansible playbook

The default login to the instance is as ec2-user , who has sudo privileges, the playbook has user set to root, we would need to modify that in both broker.yml and node.yml

```
TODO: one liner sed for this job?
```

Next we need to run the following command after changing directory to the playbook directory, replace the private key with your environment specific configuration:

```
ansible-playbook --private-key=/home/jeevanullas/sshlogin --user=ec2-user --sudo broker.yml
```

That should do it , and setup the broker.

An output from a sample run in lab is available [here](https://gist.github.com/jeevanullas/5335541#file-openshift-ansible-playbook-broker-output-txt)

Next would be the node, replace the private key with your environment specific configuration:

```
ansible-playbook --private-key=/home/jeevanullas/sshlogin --user=ec2-user --sudo node.yml
```

An output from a sample run in lab is available [here](https://gist.github.com/jeevanullas/5336280#file-openshift-ansible-playbook-node-output-txt)