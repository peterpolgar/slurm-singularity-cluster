# Slurm cluster in Singularity container

This works without root privilege, tested on a governmental HPC without root privilege.
Currently, this configuration creates only one compute node (which node is the same as the controller and the accounting manager node). If you know how to configure multiple compute nodes (like in the [slurm-docker-cluster](https://github.com/giovtorres/slurm-docker-cluster), I mean multiple virtual compute node on a same physical machine), then please create a pull request. Configuring multiple physical compute node should work, but it is not tested.

## Steps to create a slurm cluster with one compute node in a singularity container:

IMPORTANT! By default, in the slurm.conf and the slurmdbd.conf the hostname is vn01, if your host has another hostname, then replace vn01 with your hostname in the slurm.conf and the slurmdbd.conf files. You can get your hostname with the command ```echo $HOSTNAME```.

On host system:
```bash
git clone https://github.com/peterpolgar/slurm-singularity-cluster.git
cd slurm-singularity-cluster
singularity build --fakeroot slurm.sif slurm.def
# This command below creates a temporary instance, a sandbox environment, so all changes will lost when you stop the instance
singularity instance start --fakeroot --writable slurm.sif sis
singularity shell instance://sis # you can exit from and reopen the shell, no data (or changes) will loss
```

### If you immediately want to use the slurm cluster after you created the singularity instance, do this in singularity shell:

```bash
# Check if slurm initialization has been ended, it takes appr. 1 minute to complete the initialization after the singularity instance has been created
# If slurm initialization is done, then the /data/done file exists, else it does not exist.
ls /data/done
# If the output is '/data/done' then the slurm cluster is ready to use
# If the output is 'ls: cannot access '/data/done': No such file or directory', then the slurm cluster is NOT yet ready to use
```

## Check with the sinfo command (in singularity shell):

If everything is working, then the ```sinfo``` command should produce something like this:

```bash
Singularity> sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
fopart*      up   10:00:00      1   idle vn01
```


## To stop slurm (in singularity shell):

```bash
pkill slurmd
pkill slurmctld
pkill slurmdbd
```

## To stop mysqld (in singularity shell):

```bash
pkill mysqld
```

## To stop the created singularity instance (on host):

```bash
singularity instance stop sis
```
