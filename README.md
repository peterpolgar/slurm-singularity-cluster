# Slurm cluster in Singularity container(s)

This works without root privilege.

Configuring multiple physical compute nodes works, [see the setup here](#steps-to-create-a-slurm-cluster-with-multiple-physical-compute-nodes).

## Prerequisite
Installed SingularityCE version >= 4.0.1. Theoretically, the configurations below also work with unprivileged installation(s) of the SingularityCE software, it is not tested.

## Steps to create a slurm cluster with one compute node in a singularity container

This configuration creates only one compute node which node is the same as the controller and the accounting manager node. If you know how to configure multiple **virtual** compute nodes (like in the [slurm-docker-cluster](https://github.com/giovtorres/slurm-docker-cluster)), then please create a pull request.

On host system:
```bash
git clone https://github.com/peterpolgar/slurm-singularity-cluster.git
cd slurm-singularity-cluster
singularity build --fakeroot --build-arg hostname=${HOSTNAME} slurm.sif slurm.def
cat /etc/hosts > myhosts
# This command below creates a temporary instance, a sandbox environment,
#     so all changes will lost when you stop the instance
singularity instance start --fakeroot --writable -c --bind myhosts:/etc/hosts slurm.sif sis
# Check if instance initialization is ended (do not afraid of "No such file or directory" output):
xd=""; while [[ $xd != "/data/done" ]]; do sleep 1; xd=`singularity exec instance://sis ls /data/done`; done
```

## Start Singularity shell

```bash
singularity shell instance://sis # you can exit from and reopen the shell,
                                 #     no data (or changes) will loss
```

## Check with the sinfo command (in singularity shell)

If everything is working, then the ```sinfo``` command should produce something like this:

```bash
Singularity> sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
fopart*      up   10:00:00      1   idle vn01
```

## Sample jobs
```bash
Singularity> srun echo yes
yes
Singularity> srun --pty bash
root@vn01:/# ls
bin   data  environment  home  lib32  libx32  mnt  proc  run   singularity  sys  usr
boot  dev   etc          lib   lib64  media   opt  root  sbin  srv          tmp  var
root@vn01:/# exit
exit
Singularity> sacct
JobID           JobName  Partition    Account  AllocCPUS      State ExitCode
------------ ---------- ---------- ---------- ---------- ---------- --------
1                  echo     fopart       root          1  COMPLETED      0:0
1.0                echo                  root          1  COMPLETED      0:0
2                  bash     fopart       root          1  COMPLETED      0:0
2.0                bash                  root          1  COMPLETED      0:0
Singularity> 
```

## To stop slurm (in singularity shell)

```bash
pkill slurmd
pkill slurmctld
pkill slurmdbd
```

## To stop the created singularity instance (on host)

All changes made to the instance will be lost after this command:

```bash
singularity instance stop sis
```

## To stop mysqld (in singularity shell)

```bash
pkill mysqld
```

## To stop sshd (in singularity shell)

```bash
pkill sshd
```

## Steps to create a slurm cluster with multiple physical compute nodes

**Required**: SingularityCE version >= 4.0.1; These ports do not be blocked by firewall (or by other things): 6827, 6828, 6829, 6830

This section provides a description of how to set up a slurm cluster of two compute nodes, additional compute nodes can be added in a similar manner.

1. Install slurm-singularity-cluster on a machine you want to be the controller node (and the accounting manager node and a compute node) with [the steps above](#steps-to-create-a-slurm-cluster-with-one-compute-node-in-a-singularity-container).
2. On the controller node, ensure that there is an assigned ip address to the hostname of the compute node you want to add in the ```/etc/hosts``` file inside Singularity container, if no such entry, edit that file (in Singularity shell).
3. On the controller node, add the compute node to the slurm config file, e.g. add a compute node with hostname "laptop" (in Singularity shell):
```bash
echo 'NodeName=laptop State=UNKNOWN RealMemory=3000' >> /usr/local/etc/slurm.conf
```
4. Install slurm-singularity-cluster with ```slurm_compute_node.def``` on a machine you want to be a compute node:

First, download:
```bash
git clone https://github.com/peterpolgar/slurm-singularity-cluster.git
cd slurm-singularity-cluster
```
Then, configure ```slurm_compute_node.def``` with your controller's hostname
```bash
sed -i 's/vn01/controller_hostname/g' slurm_compute_node.def
```

Then,
```bash
singularity build --fakeroot slurm.sif slurm_compute_node.def
cat /etc/hosts > myhosts
```

**Important!**: If necessary, edit the ```myhosts``` file to set an ip address for the hostname of the controller node.

Finally,
```bash
# This command below creates a temporary instance, a sandbox environment,
#     so all changes will lost when you stop the instance
singularity instance start --fakeroot --writable -c --bind /dev/fuse --bind myhosts:/etc/hosts slurm.sif sis
# Check if instance initialization is ended (do not afraid of "No such file or directory" output):
xd=""; while [[ $xd != "/data/done" ]]; do sleep 1; xd=`singularity exec instance://sis ls /data/done`; done
```

5. Start slurm daemons

(in Singularity shell)

First kill slurm daemons on controller node:
```bash
pkill slurmd
pkill slurmctld
pkill slurmdbd
```
Start daemons on controller node:
```bash
slurmdbd
sleep 1
slurmctld
sleep 1
slurmd
```
Start slurmd on compute node:
```bash
slurmd
```
6. Check

(in Singularity shell)

If everything is working, then the sinfo command should produce something like this:
```bash
Singularity> sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
fopart*      up   10:00:00      2   idle laptop,vn01
```

If a compute node state is not idle, try to change it to idle with this command, e.g.:
```bash
scontrol update nodename=laptop state=idle
```
7. Test the slurm cluster with an MPI application

(in Singularity shell)

On controller node:
```bash
cd /data/shared
wget https://gist.githubusercontent.com/understeer/9462697/raw/3536b3b365b34c0af7f567b17ca1bc042ae0ef3c/mpitest.c
mpicc mpitest.c -o mpitest
# pretest the application:
mpirun --allow-run-as-root -n 2 mpitest
# the test:
salloc -N 2 mpirun --allow-run-as-root mpitest
```
On successful execution, the salloc command above should produce something similar to the following output:
```bash
salloc: Granted job allocation 4
Start! rank:0 size: 2 at vn01
Done!  rank:0 size: 2 at vn01
Start! rank:1 size: 2 at laptop
Done!  rank:1 size: 2 at laptop
salloc: Relinquishing job allocation 4
```

Note: If you want to unmount the shared mounted folder on compute node in Singularity container:
```bash
umount /data/shared
```
