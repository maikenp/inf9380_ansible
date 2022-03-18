# PLAYBOOKS - HOW-TO

## install_slurm.yml



### Prerequisites: 

Assumes RHEL7/8 compatible system.


#### Master node:

Firewall:

- Must have ports 6817 open for slurmctld - compute-nodes will contact the master node via this port 
- If slurmdbd is running, port 6819 must be open for slurmdbd


#### Compute nodes:


Firewall:

- The computes must all be able to talk to each other over all ports. 
- In particular port 6818 must be open for slurmd



### Preparations for running ansible:



#### 1) Create an ansible inventory file:



Create a hosts file,  e.g. /etc/ansible/hosts using the group-names and format as below example:

```
[master]
frontend01   ansible_host=10.2.3.4  

[compute]
compute-c01 ansible_host=10.2.1.174
compute-c02 ansible_host=10.2.1.160

[cluster]
frontend01   ansible_host=10.2.3.4  
compute-c01 ansible_host=10.2.1.174
compute-c02 ansible_host=10.2.1.160
```` 

Obviously change the hostnames and ip-adresses according to your cluster.



#### 2) Edit the default variables in the playbook:


Especially these variables need editing

- slurm_cluster_name
  - The name you chose to give the cluster - will show up in slurm.conf
  
- SLURM_ACCOUNTING_DB_PASS

  - mariadb password
- slurm_submitters 

   - if slurmdbd is set up, give a list of users that are allowed to submit to slurm. In the example defaults/main.yml only one submitter is added, just add another item in the list if needed

- install_local
  - Install using pre-built local rpms or using the EPEL repo.
  - Allowed values are yes or no
  
- os_v
  - OS version
  - Allowed values are el7 or el8

- slurm_v
  - Slurm version - if installed locally then you have to build the rpms and place them in the files/slurmrpms first - se details below for instructions
 
- munge_v
  - Munge version - same comment as above




### Running the playbook
Ansible just uses ssh to connect to the remote servers. If your method to connect to the remote server is via password and normal user, and that you need to enter sudo password for privelege escalation, then run ansible in the following way:

Run the commands below in the same directory as the playbook file for simplicity. 

```
ansible-playbook -i <path-to-your-ansible-inventory-file> install_slurm.yml  --ask-pass --ask-become-pass
```

For example:

````
ansible-playbook -i /etc/ansible/hosts install_slurm.yml --ask-pass --ask-become-pass
````

This will install, configure and start slurmctld, slurmdbd and mariadb on the master and slurmd on the compute-nodes. 
In addition it will install a healthcheck script which is not customized for general use. 
Either customize it in ./roles/slurm_compute/files/node_healthcheck or skip it. 

For installation without accounting, no slurmdbd nor mariadb is needed. Here also skippint the healthcheck and jobplugin task

```
ansible-playbook -i /etc/ansible/hosts install_slurm.yml --ask-pass --ask-become-pass --skip-tags "db,healthcheck,jobplugin"
```

If you. need to run the tasks for master and compute separately - for instance needed if one machine is NREC with ssh keys and passwordless sudo acces
but the other is a uio machine with user-name password and not passwordless sudo access you will run the playbook twice, here shown with compute as needing password and master not

To run

     ansible-playbook -i /etc/ansible/hosts -l compute install_slurm.yml  --skip-tags "db,healthcheck,jobplugin"  --ask-pass --ask-become 
     ansible-playbook -i /etc/ansible/hosts -l master install_slurm.yml --skip-tags "db,healthcheck,jobplugin"


#### ssh stuff
Otherwise, you can set up the remote host to allow logging in without password by injecting your public key into the remote host authorized_keys file. 
Details are left for you to find out via google and/or know-how of ssh. (E.g. here: https://code-maven.com/enable-ansible-passwordless-sudo) 


### Testing the slurm installation
Log into the master/login node or one of the compute-nodes to submit a test-job. If you have set up slurm database, the user you submit with must be allowed to submit jobs (check the slurm_submitters variable in default/main.yml)

save this to a file submitslurm.sh:

``` 
#!/bin/bash
#
#SBATCH --job-name=test
#
#SBATCH --ntasks=1
#SBATCH --time=10:00
#SBATCH --mem-per-cpu=100
#SBATCH --output=/tmp/slurmout.txt

srun echo hostname > /tmp/slurmout.txt
srun sleep 60
``` 

Then submit the jobs with

``` sbatch submitslurm.sh ```

You should see output with your slurm job-id, e.g.: 

```
Submitted batch job 5
``` 


Check the job with using the jobid you got from the step above:
```
scontrol show job <jobid>
```



### What does the playbook do


Slurm can be run with or without a database. If the setup is simple, and there is no need for accounting, you can run without slurm database. 

#### Basics 


slurmctld, slurmd, and munge installation and configuration

- Prepare all folders needed for slurm on cluster
- Create slurm.conf and slurmnodes.conf based on templates on cluster
- Create slurm linux user on cluster
- Create munge linux user on cluster
- Install munge on cluster
- Create munge key on master, and copy key to compute-nodes
- On master: install slurmctld 
- On compute-nodes: install slurmd 
- slurmctld runs as slurm user (SlurmUser variable in slurm.conf)
- slurmd runs as root user (SlurmdUser variable in slurm.conf)


#### DB-installation

If slurmdbd is to be installed: slurmdbd, mariadb installation and configuration

- Create slurm-submitter linux users on cluster
- Install mariadb and MySQL-python on master node
- Configure innodb settings for mariadb
- Install slurmdbd on master node
- Create mysqlDB user for slurmdbd
- Add cluster, accounts and users to slurmdbd on master node
- slurmdbd runs as slurm user on master node (SlurmUser variable in slurmdbd.conf)


### Build slurm and munge rpms
https://slurm.schedmd.com/quickstart_admin.html


    dnf install epel-release -y
    dnf --enablerepo=powertools install wget rpm-build bzip2-devel openssl-devel zlib-devel gcc readline-devel pam-devel perl-ExtUtils-MakeMaker perl-DBI mysql-devel make munge-devel python3 lua-devel -y  
    wget https://github.com/dun/munge/releases/download/munge-0.5.14/munge-0.5.14.tar.xz.asc
    wget  https://github.com/dun/munge/releases/download/munge-0.5.14/munge-0.5.14.tar.xz
    wget https://github.com/dun.gpg 
    rpmbuild -tb munge-0.5.14.tar.xz
    rpmbuild -tb --clean --rmsource --without debug --with lua slurm-20.11.3.tar.bz2

Place the rpms produed in the roles/slurm/files/slurmbuild/ folder.
Make sure to hange the slurm_v and munge_v to match the version you built.


