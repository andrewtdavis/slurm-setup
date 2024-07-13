# SLURM Installation & Configuration
Setting up SLURM for advanced, Enterprise Linux based GPU clusters

## Database and UID-Synced Users
_This assumes the SLURM head node is a VM with a separate 200G disk for SLURM accounting database in MariaDB_
1. Setup data disk
   ```shell
   pvcreate /dev/sdb
   vgcreate slurm /dev/sdb
   lvcreate -l 100%FREE -n database slurm
   mkfs.xfs /dev/mapper/slurm-database
   ```
2. Add to `/etc/fstab`
   ```shell
   /dev/mapper/slurm-database /var/lib/mysql       xfs     noatime,defaults 0 0
   ```
3. Install the MariaDB database on the head node only
   ```shell
   dnf install mariadb-server mariadb-devel
   ```
4. Setup uidNumber-matched Munge and SLURM users across all nodes in the cluster.
   1. `groupadd -g ##### munge`
   2. `groupadd -g ##### slurm`
   3. `useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u##### -g munge -s /sbin/nologin munge`
   4. `useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u##### -g slurm -s /bin/bash slurm`
5. Create file `/etc/my.cnf.d/innodb.cnf`, with contents:
   ```text
   [mysqld]
   # Set to 50% system RAM
   innodb_buffer_pool_size=6G
   innodb_log_file_size=64M
   innodb_lock_wait_timeout=900
   ```
6. Start and enable the MariaDB server
    ```shell
    systemctl enable mariadb; systemctl start mariadb; systemctl status mariadb
    ```
7. Setup MariaDB with a secure configuration:
   ```shell
   /usr/bin/mysql_secure_installation
   ```
8. Setup SLURM user & Database

   ⚠️ WARNING: change mypassword to something secure
```sql
grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by 'mypassword' with grant option;
SHOW GRANTS;
SHOW VARIABLES LIKE 'have_innodb';
create database slurm_acct_db;
```

## EPEL
_Many packages for SLURM and CUDA require the EPEL repository_
- RHEL 8:
   ```shell
   subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms
   dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
   ```
- Rocky 8:
   ```shell
   dnf config-manager --set-enabled powertools
   dnf install epel-release
   ```
- RHEL 9:
   ```shell
   subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
   dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
   ```
- Rocky 9:
   ```shell
   dnf config-manager --set-enabled crb
   dnf install epel-release
   ```
## NFS Mounts
_While SLURM doesn't explicitly require synced home directories, users will likely call scripts in their homes. Therefore, the reality is homes must be in sync across the cluster. While there are clustered filesystems that are commonly used with HPC like Lustre or GPFS, NFS is a no-cost alternative_

1. Install NFS & autofs:
   ```shell
   dnf install nfs-utils autofs
   ```
      - `autofs` is not required, but can be useful to limit mounted homes and shares but is not required.
2. Create and mount a cluster home mountpoint, such as `/cluster/home`
3. Under the `home`, add a new one for `root` if it doesn't exist.
4. Ensure that other has no access
5. symlink it to /root/work on all nodes
6. Create some basic directories:
   ```shell
   mkdir work/autofs work/hpc-x work/munge work/modulefiles work/slurm work/slurm/conf work/slurm/rpms work/slurm/source
   ```

## Required Packages
- SLURM Queue Master or other build host
   ```shell
   dnf groupinstall "Development Tools"
   sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
   dnf clean expire-cache
   dnf install cuda-toolkit-12-2
   ```
  - To enable GPU scheduling, CUDA must be installed and enabled. Note: CUDA can be installed without a GPU present.
  - Add to root's `.bashrc`
      ```text
      # Load CUDA 12.2
      export PATH=/usr/local/cuda-12.2/bin:$PATH
      export LD_LIBRARY_PATH=/usr/local/cuda-12.2/lib64:$LD_LIBRARY_PATH
      export CUDA_LIB_PATH=/usr/local/cuda-12.2/lib64
      ```
  - To enable Infiniband-based multi-node MPI workloads, install and load [HPC-X](https://docs.nvidia.com/networking/display/hpcxv2171/installing+and+loading+hpc-x)
    - Add to root's `.bashrc`, changing the path to your version
        ```text
        # Load HPC-X
        source /root/hpcx-v2.18-gcc-inbox-redhat8-cuda12-x86_64/hpcx-init.sh
        hpcx_load
        ```
  - To enable IP-based multi-node MPI workloads, install `openmpi` and ensure its binaries are in your path (they're not by default)
- All nodes:
   ```shell
   dnf install wget make gcc bzip2-devel libffi-devel zlib-devel python39 htop tmux nfs-utils openssl openssl-devel pam-devel numactl numactl-devel hwloc hwloc-devel lua lua-devel readline-devel rrdtool-devel ncurses-devel libibmad libibumad rpm-build munge-devel perl-devel python3 infiniband-diags gtk2-devel libbpf libbpf-devel dbus-devel freeipmi-devel
   ```

## Munge
_Munge is the authentication system SLURM uses to communicate between hosts_
1. On all cluster nodes:
   ```shell
   dnf install munge
   ```
2. On first node: build the cluster's secret key
   ```shell
   dd if=/dev/urandom bs=1 count=1024 >work/munge/munge.key
   cp work/munge/munge.key /etc/munge
   chown munge:munge /etc/munge/munge.key
   chmod 400 /etc/munge/munge.key
   ```
3. On all other nodes: Copy and secure the key:
   ```shell
   cp work/munge/munge.key /etc/munge/
   chown munge:munge /etc/munge/munge.key
   chmod 400 /etc/munge/munge.key
   ```
4. Start and enable the munge daemon:
   ```shell
   systemctl start munge;systemctl enable munge;systemctl status munge
   ```

## Build SLURM RPMs
1. Make a build directory on the head node node at a shared NFS path:
   ```shell
   cd /root/work/slurm/source/
   ```
2. Download the latest [SLURM](https://www.schedmd.com/download-slurm/) into the build directory
3. Build the RPMs from the archive:
   ```shell
   rpmbuild -ta --with freeipmi slurm*.bz2
   ```
4. Copy the RPMs back to the slurm directory
   ```shell
   cp -r /root/rpmbuild/RPMS/x86_64 work/slurm/rpms/
   ```

## Install SLURM across the cluster
From the `work/slurm/rpms/$arch` path, install SLURM depending on node type:
- Head node:
   ```shell
   dnf install slurm-24*.rpm slurm-perlapi-*.rpm slurm-slurmctld-*.rpm slurm-slurmdbd-*.rpm slurm-libpmi-*.rpm slurm-devel*rpm
   ```
- Compute nodes:
   ```shell
   dnf install slurm-24*.rpm slurm-perlapi-*.rpm slurm-slurmd-*.rpm slurm-libpmi-*.rpm slurm-devel*rpm
   ```
- Login / submit nodes:
```shell
dnf install slurm-24*.rpm slurm-perlapi-*.rpm slurm-libpmi-*.rpm slurm-devel*rpm
```

## Setup SlurmDBD
_SlurmDBD is Slurm's Database daemon, which is responsible for communicating between one or more slurmctld hosts and a back-end MySQL-compatible database._
1. Make base slurm config directory:
   ```shell
   mkdir /etc/slurm
   ```
2. Create `/etc/slurm/slurmdbd.conf`
   <details>
      <summary>Example SlurmDBD Configuration</summary>
   
   Make sure to change:
   - `DbdAddr` to the IP address of your SlurmDBD host
   - `DbdHost` to the DNS hostname of your SlurmDBD host
   - `StoragePass` to the MariaDB password setup above
      ```text
      # Archive info
      #ArchiveJobs=yes
      #ArchiveDir="/tmp"
      #ArchiveSteps=yes
      #ArchiveScript=
      #JobPurge=12
      #StepPurge=1
      #
      # Authentication info
      AuthType=auth/munge
      #AuthInfo=/var/run/munge/munge.socket.2
      #
      # slurmDBD info
      DbdAddr=X.X.X.X # Change to the IP of the SlurmDBD Host
      DbdHost=X.example.com # Change to the DNS name of the SlurmDBD Host
      #DbdPort=7031
      SlurmUser=slurm
      #MessageTimeout=300
      DebugLevel=verbose
      #DefaultQOS=normal,standby
      LogFile=/var/log/slurm/slurmdbd.log
      PidFile=/var/run/slurmdbd.pid
      #PluginDir=/usr/lib/slurm
      #PrivateData=accounts,users,usage,jobs
      #TrackWCKey=yes
      #
      # Database info
      StorageType=accounting_storage/mysql
      StorageHost=localhost
      StoragePort=6819
      StoragePass=mypassword # Change the password to the MySQL user setup above
      StorageUser=slurm
      StorageLoc=slurm_acct_db
      #
      # Purge old records after 90 days
      PurgeEventAfter=90days
      PurgeJobAfter=90days
      PurgeResvAfter=90days
      PurgeStepAfter=90days
      PurgeSuspendAfter=90days
      PurgeTXNAfter=90days
      PurgeUsageAfter=90days
      ```
   </details>
3. Restrict access to the config file, as it has a DB password in it:
   ```shell
   chown slurm /etc/slurm/slurmdbd.conf
   chmod 600 /etc/slurm/slurmdbd.conf
   ```
4. Test the database connection:
   ```shell
   slurmdbd -D
   ```
5. If everything looks good, CTRL-C and start the service proper:
   ```shell
   systemctl enable slurmdbd; systemctl start slurmdbd
   ```

## Build & Deploy Configuration Files
1. Create `work/slurm/conf/slurm.conf`, editing the following lines:
   - `ClusterName` Set to the name of the cluster 
   - `SlurmctldHost` Set to the short-name (`hostname -s`) of the slurm head node
   - `AccountingStorageHost` Set to the short-name (`hostname -s`) of the slurm head node
   - `AccountingStorageTRES` Uncomment and set to the GPU(s) that will be tracked. Only for GPU clusters
   - `GresTypes` Uncomment if using GPU clusters
   - `SelectTypeParameters` Set to `CR_Core` to allow jobs to share nodes (default:exclusive per job), and `CR_Pack_Nodes` (to keep jobs on as few nodes when large jobs are frequent) or `CR_LLN` to distribute jobs across nodes (useful for network-heavy workloads).
   <details>
      <summary>Example slurm.conf file</summary>
   
   ```text
   # Slurm Configuration
   ClusterName=CHANGEME
   SlurmctldHost=CHANGEME-q1(X.X.X.X)
   #
   #MailProg=/bin/mail
   MpiDefault=none
   #MpiParams=ports=#-#
   ProctrackType=proctrack/cgroup
   ReturnToService=1
   SlurmctldPidFile=/var/run/slurmctld.pid
   #SlurmctldPort=6817
   SlurmdPidFile=/var/run/slurmd.pid
   #SlurmdPort=6818
   SlurmdSpoolDir=/var/spool/slurmd
   SlurmUser=slurm
   #SlurmdUser=root
   StateSaveLocation=/var/spool/slurmctld
   SwitchType=switch/none
   TaskPlugin=task/affinity,task/cgroup
   LaunchParameters=use_interactive_step
   #
   #
   # TIMERS
   #KillWait=30
   #MinJobAge=300
   #SlurmctldTimeout=120
   #SlurmdTimeout=300
   #
   #
   # SCHEDULING
   SchedulerType=sched/backfill
   SelectType=select/cons_tres
   SelectTypeParameters=CR_Core
   #
   #
   # LOGGING AND ACCOUNTING
   #
   # Use the Slurm Database Daemon
   AccountingStorageType=accounting_storage/slurmdbd
   AccountingStorageHost=CHANGEME
   # How to gather job data
   JobAcctGatherType=jobacct_gather/cgroup
   # Gather GPU usage
   # Set depending on GPUs to monitor:
   # A100: gres/gpu:a100
   # H100: gres/gpu:h100
   # Generic GPU request: gres/gpu
   #AccountingStorageTRES=gres/gpu,gres/gpu:h100,gres/gpu:a100
   # How often to gather job data
   JobAcctGatherFrequency=task=30,energy=30,network=30,filesystem=30
   # Set limits based on accoutning DB
   # WARNING: This requires setting up SLURM accounts in the database, and mapping users to those accounts for jobs to launch.
   #AccountingStorageEnforce=limits
   #SlurmctldDebug=info
   SlurmctldLogFile=/var/log/slurm/slurmctld.log
   #SlurmdDebug=info
   SlurmdLogFile=/var/log/slurm/slurmd.log
   #
   #
   # Setup gres for GPUs
   # GresTypes=gpu
   #
   #
   # NODE CONFIGURATION
   # Nodes go here. Configuration lines can be found by running 'slurmd -C' on a compute node. Make sure to adjust cores & memory down to account for kernel and any system processes.
   #
   # PARTITION CONFIGURATION
   # Partitions go here
   # Examples (set the DefMemPerCPU to match compute nodes: RealMemory/cores):
   # PartitionName=interactive Nodes=compute-node[01-10] Default=YES PriorityJobFactor=100 State=UP DefMemPerCPU=1940
   # PartitionName=batch Nodes=compute-noe[01-10] PriorityJobFactor=1 State=UP DefMemPerCPU=1940
   ```
   </details>
2. For each type of node, run `slurmd -C` on the host to get the node config
 If you're setting up multiple identical nodes that will be named sequentially, then one can be added as a range. Example:
   ```text
   NodeName=node[01-03] CPUs=64 Boards=1 SocketsPerBoard=2 CoresPerSocket=32 ThreadsPerCore=2 RealMemory=1024000
   ```
3. Setup the Partition(s)
   - Example basic partition:
     ```text
     PartitionName=default Nodes=ALL Default=YES State=Up
     ```
   - Advanced GPU partition:
   ```text
   PartitionName=dev Nodes=labnode01 MaxTime=16:00:00 PriorityJobFactor=100 AllowGroups=mylab State=UP DefMemPerCPU=16000 DefCpuPerGPU=16 QOS=MaxJobsPerUser=1
   ```
   - ℹ️ For GPU partitions, set `DefMemPerCPU` & `DefCpuPerGPU` with the following format:
     - `DefMemPerCPU` = `RealMemory` / `CPU`, round down
     - `DefCpuPerGPU` = `CPU` / `# GPU`, round down
   - Other options to consider:
     - `MaxTime` Time in the format of D-HH:MM:SS that jobs can request, to limit run length
     - `PriorityJobFactor` Set the priority to cause some partitions' jobs to run before others. For example interactive queues can take priority 
     - `AllowGroups` Set the groups (comma separated) to gate specific partitions to specific linux groups 
     - `MaxNodes` Set the maximum number of nodes a specific job can request.
4. Copy the config to all Slurm nodes (head node, compute nodes, login nodes)
   ```shell
   mkdir /etc/slurm
   cp work/slurm/conf/slurm.conf /etc/slurm
   ```
5. Setup `work/slurm/conf/cgroup.conf` with the following contents:
   ```text
   # Limit job cores to requested number
   ConstrainCores=yes
   
   # Limit RAM to 100% of requested. May OOM kill job processes if the overall usage goes above the requested number
   AllowedRAMSpace=100
   ConstrainRAMSpace=yes
   
   # UN-COMMENT IF USING GPUS:
   # Limit GPU device access to requested GPUs
   # ConstrainDevices=yes
   ```
6. Copy this to all compute nodes:
   ```shell
      cp work/slurm/conf/cgroup.conf /etc/slurm
   ```
7. Setup folders on the head node:
   ```shell
   mkdir /var/spool/slurmctld
   chown slurm: /var/spool/slurmctld
   chmod 755 /var/spool/slurmctld
   ```
8. Setup folders on compute and login/submit nodes:
   ```shell
   mkdir /var/log/slurm
   chown slurm /var/log/slurm
   ```

## Firewall Configuration
_Depending on environment, restricting cluster communication to known hosts can be a good safety net while preventing communication issues within the clsuter_

Add firewall exceptions for intra-cluster communication, depending on RHEL version
- Recommended: Create a list of commands that contain every node in the cluster, and then run it on every node. It's fine if a host has its own IP in the allowlist. 
- **RHEL/Rocky 8**: Add rich rules for each node, and then reload the config.
   ```shell
   firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="X.X.X.X" accept'
   firewall-cmd --reload
   ```
- **RHEL/Rocky 9**: Create a new trusted zone for cluster hosts to allow all traffic.
   ```shell
   firewall-cmd --zone=trusted --add-source=X.X.X.X
   firewall-cmd --runtime-to-permanent
   ```

## Start & Test Slurm
1. Enable and start the Slurm master on the head node:
   ```shell
   systemctl enable slurmctld.service; systemctl start slurmctld.service; systemctl status slurmctld.service
   ```
2. Enable and start the Slurm daemons on all compute hosts:
```shell
systemctl enable slurmd.service; systemctl start slurmd.service; systemctl status slurmd.service
```
3. Test the cluster (# is the number of compute hosts in the cluster)
   ```shell
   srun -N# /bin/hostname
   ```
4. Check cluster status:
   ```shell
   scontrol show nodes
   scontrol show partition
   scontrol show jobs
   ```