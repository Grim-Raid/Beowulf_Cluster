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
1. Create new user called mpiuser under System Settings>Users and Groups>Add account. Make sure this account is an admin on the device. From now on, all commands should be run through the mpiuser. To switch users run
```zsh
su - mpiuser
```
3. Go to System Settings>General>Sharing and enable remote login (this allows other devices to ssh into the device)
4. create ssh keys for all devices on manager run the following and follow the next prompts.
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

5. Setup NFS, on manager node, make cloud directory
```zsh
mkdir cloud
```
then run
```zsh
sudo nano /etc/exports
```
add
```
/Users/mpiuser/cloud -maproot=mpiuser WorkerIPAddresses
```
then run 
```zsh
sudo nano /etc/hosts
```
add
```
manager MANAGERIPADDRESS
worker1 WORKER1IPADDRESS
worker2 WORKER2IPADDRESS
```
then run
```zsh
nfsd restart
```
to check that it is properly set up run
```zsh
showmount -e
```
On worker nodes, run 
```zsh
sudo nano /etc/hosts
```
add
```
manager MANAGERIPADDRESS
worker1 WORKER1IPADDRESS
```
then run
```zsh
sudo mount -t nfs manager:/home/mpiuser/cloud ~/cloud
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
managerIPaddress slots=4 max_slots=40
worker1IPaddress slots=4 max_slots=40
worker2IPaddress  max_slots=40
worker3IPaddress slots=4 max_slots=40
```

## compiling PHITS for MPI on MacOS
use the following the in ?make file? of the PHITS source code 
```c
ENVFLAG = MacGFort
### Mac gfortran (4.8, 7.0 or later)


ifeq ($(ENVFLAGS),MacGfort)
 OtherGfortran = true
 SRCS8 += mdp-uni90.f
 FC  = mpifort  # you have to revise the version
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
endif
```
