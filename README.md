# Slurm cluster in Singularity container

This works without root privilege, tested on a governmental HPC without root privilege.
Currently, this configuration creates only one compute node (which node is the same as the controller and the accounting manager node). If you know how to configure multiple virtual compute nodes (like in the [slurm-docker-cluster](https://github.com/giovtorres/slurm-docker-cluster)), then please create a pull request. Configuring multiple physical compute node works, it is tested, details are coming soon...

## Steps to create a slurm cluster with one compute node in a singularity container

On host system:
```bash
git clone https://github.com/peterpolgar/slurm-singularity-cluster.git
cd slurm-singularity-cluster
singularity build --fakeroot --build-arg hostname=${HOSTNAME} slurm.sif slurm.def
# This command below creates a temporary instance, a sandbox environment,
#     so all changes will lost when you stop the instance
singularity instance start --fakeroot --writable -c slurm.sif sis
# Check if instance initialization has ended (do not afraid of "No such file or directory" output):
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

