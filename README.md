# Beowulf_Cluster

## Introduction
This repository contains the information on how to implement a beowulf cluster over MPI on Apple Silicon. There are ways to adapt this for interchange between ARM (Apple Silicon) and the x86 (Intel) Archetectures, but that causes unwanted errors and makes things difficult due to Rosetta. This contain a step by step guide on how to create a beowulf cluster

## Prerequisites

* **[Homebrew](brew.sh) must be installed on all nodes**
* **All Firewalls need to be disabled on all devices**
* Static IPs on all devices makes things easier, otherwise after a power cycle the hosts files will need to be changed to new IPs.
* At least two apple silicon devices you want to cluster


## Cluster archetecture
Whenever the cluster is set up, one device will act as the "manager" node, and all other devices with act as "worker" nodes. Whenever the manager does certain tasks for setup, the manager must do this for all worker IPs. 
## Setup
1. install open mpi by running `brew install openmpi`
1. Create new user called mpiuser under System Settings>Users and Groups>Add account. Make sure this account is an admin on the device
1. Go to System Settings>General>Sharing and enable remote login (this allows other devices to ssh into the device)
1. create ssh keys for all devices on manager run the following and follow the next prompts.
```zsh
ssh-keygen -t rsa
cd .ssh/
cat id_rsa.pub >> authorized_keys
ssh-copy-id ipaddressofworker
```

This must be done for all workers on the manager node
on worker nodes run 
```zsh
ssh-keygen -t rsa
cd .ssh/
cat id_rsa.pub >> authorized_keys
ssh-copy-id ipaddressofmanager
```

1. Setup NFS, on manager node, run
```zsh
sudo nano /etc/exports
```
add
```
***WIP because I forgot that MacOS uses dumb syntax, need to check on mac studio in lab***
/Users/mpiuser/cloud workeripaddresss(rw,sync,no_root_squash,no_subtree_check) 
```
then run
```zsh
nfsd restart
```
On worker nodes, run 
```zsh
sudo mount -t nfs manageripaddress:/home/mpiuser/cloud ~/cloud
```
check by running `df -h` on workers

Note: need to figure out how to run these scripts on init, else have to remount storage after every reboot on all devices

## Running MPI Programs
Running MPI programs is easy just use ```mpirun```
For example to run mpi_hello
```zsh
mpirun -hostfile my_host ./mpi_hello
```
Where my_host is a file that looks like
```zsh
managerIP slots=4 max_slots=40
worker1IP slots=4 max_slots=40
worker2IP  max_slots=40
worker3IP slots=4 max_slots=40
```

## compiling PHITS for MPI on MacOS
use the following the in ?make file? of the PHITS source code 
```[6/22 1:22 PM] Gregory Field

using brew install: openmpi

use ENVFLAG = MacGFort

 

### Mac gfortran (4.8, 7.0 or later)


ifeq ($(ENVFLAGS),MacGfort)


 OtherGfortran = true


 SRCS8    += mdp-uni90.f


 FC        = mpifort  # you have to revise the version


 FFLAGS    = -O0 -fdefault-double-8 -fdefault-real-8 -fdollar-ok -std=legacy


 ifeq ($(USEMPI),true)


  FFLAGS  +=   $(shell mpifort -showme:compile)


  FFLAGS  += -I$(shell mpifort -showme:incdirs)


  LDFLAGS  =   $(shell mpifort -showme:link)


  LDLIBS   = -lmpi_usempi -lmpi_mpifh -lmpi


 endif


 ifeq ($(USEOMP),true)


  FFLAGS += -fopenmp


 endif


endif```