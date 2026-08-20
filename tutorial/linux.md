# Install batching system

- **LSF**: IBM’s commercial batch system; command examples include `bsub`, `bjobs`, `bkill`.
- **Slurm**: open-source workload manager; command examples include `sbatch`, `squeue`, `scancel`

We would use **Slurm** on Ubuntu. Ubuntu 24.04 provides slurm-wlm directly from the package repository.


1. Install packages

```
sudo apt update
sudo apt install -y munge slurm-wlm
```

2. Configure Munge authentication

```
sudo systemctl enable --now munge
munge -n | unmunge
```

If `munge -n | unmunge` works, Munge is fine.

3. Get Slurm hardware config

```
hostname -s
slurmd -C
```


You should see something like:

```
NodeName=miaolab CPUs=160 Boards=1 SocketsPerBoard=1 CoresPerSocket=80 ThreadsPerCore=2 RealMemory=257406
UpTime=2-02:20:41
```

Copy that `NodeName=...` line.


4. Create `/etc/slurm/slurm.conf`

```
sudo nano /etc/slurm/slurm.conf
```

5. Create spool/log directories

```
sudo mkdir -p /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
sudo chown -R slurm:slurm /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
```

6. Configure cgroups

```
sudo nano /etc/slurm/cgroup.conf
```

Add 

```
CgroupPlugin=autodetect
ConstrainCores=yes
ConstrainRAMSpace=yes
ConstrainDevices=yes
```

7. Start Slurm

```
sudo systemctl enable --now slurmctld
sudo systemctl enable --now slurmd
```

Check status:

```
systemctl status slurmctld --no-pager
systemctl status slurmd --no-pager
```






# File edit

## nano

```
Ctrl + O
Enter
Ctrl + X
```



ClusterName=miaolab
SlurmctldHost=miaolab

MpiDefault=none
ProctrackType=proctrack/cgroup
ReturnToService=1

SlurmctldPidFile=/run/slurmctld.pid
SlurmdPidFile=/run/slurmd.pid
SlurmdSpoolDir=/var/spool/slurmd
StateSaveLocation=/var/spool/slurmctld

SwitchType=switch/none
TaskPlugin=task/cgroup,task/affinity

SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

SchedulerType=sched/backfill
AccountingStorageType=accounting_storage/none
JobCompType=jobcomp/none

# Replace this with your actual output from: slurmd -C
NodeName=miaolab CPUs=160 Boards=1 SocketsPerBoard=1 CoresPerSocket=80 ThreadsPerCore=2 RealMemory=257406

PartitionName=compute Nodes=miaolab Default=YES MaxTime=7-00:00:00 State=UP


# Run R at background

```
cd /mnt/data1/Users/hyt001/Project11/Data/Address/batch/
nohup R CMD BATCH ./address_text_1_100.R &
```

```{r}
options(scipen = 99)
# Generate linux code
for (i in 1:25) {
  cat(glue("nohup R CMD BATCH /mnt/data1/Users/hyt001/Project11/Data/Address/batch/address_text_{(i)*10000 + 1}_{(i + 1)*10000}.R &\n\n"))
}
```

# Conda

```
conda create -n py312 python=3.12 ipykernel jupyter anaconda

```

Make `conda` available for every user

```
echo 'export PATH="/mnt/data1/Software/Anaconda/bin:$PATH"' > /etc/profile.d/conda.sh
chmod +x /etc/profile.d/conda.sh
```