# Slurm cluster in Singularity container

This works without root privilege, tested on a governmental HPC without root privilege.
Currently, this configuration creates only one compute node (which node is the same as the controller and the accounting manager node). If you know how to create multiple compute nodes (like in the [slurm-docker-cluster](https://github.com/giovtorres/slurm-docker-cluster)), then please create a pull request.

## Steps to create a slurm cluster with one compute node in a singularity container:

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
