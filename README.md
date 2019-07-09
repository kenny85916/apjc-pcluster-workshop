# APJC Tech Summit - ParallelCluster Workshop

## Goal of the workshop

We will use AWS ParallelCluster to deploy a HPC cluster and run a few workloads.

AWS ParallelCluster documentation is available here: https://docs.aws.amazon.com/parallelcluster/latest/ug/

  * Part0: Let's prepare the deployment environment
  * Part1: We will start hands-on by deploying a first ParallelCluster environment.
  * Part2: While it will be deploying (15 min), I'll tell you more about ParallelCluster and it's implementation details.
  * Part3: Hands-on again, we will compile and run a first MPI application, and play with the scheduler.
  * Part4: Let's be serious, now we'll deploy a C5n cluster with EFA and FSx for Lustre.
  * Part5: While it will be deploying, I'll tell you more about EFA and FSx for Lustre.
  * Part6: Let's run a few benchmarks and play with Lustre
  * Part7: Wrap-up


## Part 0 - Preparing the deployment environment

### Pre-requisites

You will need a pre-configured EC2 key.

You will need to select a region, ideally one where you can find C5n (see part4), because that will be useful later.

### Setup an instance as a deployment system

  * Launch a T3.micro with the latest AMZL2 AMI - leave everything by default and put it in a security group that you can access using SSH (tcp/22)
  * Give it an IAM role with AdminitratorAccess

Once the instance has booted and you are logged in:

```bash
$ sudo yum update
$ sudo yum install tmux tree python3-pip
$ tmux
$ pip3 install --user aws-parallelcluster
```
## Part 1 - Deploying a simple cluster instance

Now we have ParallelCluster CLI installed and we can configure it to deploy our first cluster:

```bash
$ export AWS_DEFAULT_REGION=eu-west-1 # setup your preferred region
$ pcluster configure
Cluster Template [default]:
Acceptable Values for AWS Region ID:
    eu-north-1
    ap-south-1
    eu-west-3
    eu-west-2
    eu-west-1
    ap-northeast-2
    ap-northeast-1
    sa-east-1
    ca-central-1
    ap-southeast-1
    ap-southeast-2
    eu-central-1
    us-east-1
    us-east-2
    us-west-1
    us-west-2
AWS Region ID []: eu-west-1
VPC Name [public]:
Acceptable Values for Key Name:
    your_key
Key Name []: your_key
Acceptable Values for VPC ID:
    vpc-12345678
VPC ID []: vpc-12345678
Acceptable Values for Master Subnet ID:
    subnet-123b1234
    subnet-123c1234
    subnet-123d1234
Master Subnet ID []: subnet-123b1234
$
```

ParallelCluster configuration file is stored in ```~/.parallelcluster```:

```bash
$ tree .parallelcluster/
.parallelcluster/
├── config
└── pcluster-cli.log
```

Before deploying a first cluster, we will modify ```~/.parallelcluster/config```to suit our needs. We will modify as the exemple below, so that:
  * The scheduler will be SLURM (https://slurm.schedmd.com/)
  * We'll change the instance types to:

    * t3.xlarge for the management node
    * c5.4xlarge for the compute nodes

  * We'll set a min/max number of compute instances
  * the operating system will be set to amazon linux
  * We'll configure compute nodes so that they expose cores instead of hyperthreads


Once modified with this parameters, the configuration will look like:

```
[aws]
aws_region_name = <your_preferred_region>

[cluster default]
key_name = <your_key>
vpc_settings = public
scheduler = slurm                   <--- Here we set the scheduler
master_instance_type = t3.xlarge    <--- Master instance type
compute_instance_type = c5.4xlarge  <--- Compute instance type
initial_queue_size = 2              <--- Initial number of compute nodes
max_queue_size = 10                 <--- Max number of compute nodes
maintain_initial_size = true        <--- Min number of compute nodes = initial size
cluster_type = ondemand             <--- Compute nodes will come from on-demand market
extra_json = { "cluster" : { "cfn_scheduler_slots" : "cores"} }   <--- expose cores instead of HyperThreads
base_os = alinux                    <--- operating system = Amazon Linux

[vpc public]
vpc_id = vpc-xxxxxxx
master_subnet_id = subnet-xxxxxxx

[global]
cluster_template = default
update_check = true
sanity_check = true

[aliases]
ssh = ssh {CFN_USER}@{MASTER_IP} {ARGS}
```

Our cluster's configuration is now ready, we can start the deployment:

```bash
$ pcluster create apjc1
Beginning cluster creation for cluster: apjc1
Creating stack named: parallelcluster-apjc1
Status: parallelcluster-apjc1 - CREATE_COMPLETE
MasterPublicIP: 63.34.46.177
ClusterUser: ec2-user
MasterPrivateIP: 172.31.25.72
```

That part takes up to 15 minutes.

## Part 2 - How is it done under the cover

Now it is time for a quick presentation.

## Part 3 - Playing with your first cluster

Deployment is done, now let's play with our new toy.

First, let's connect to the cluster from the deployment system:

```bash
pcluster ssh apjc1
The authenticity of host '63.34.46.177 (63.34.46.177)' can't be established.
ECDSA key fingerprint is SHA256:5gOm8OQ6ych5sxiw0xSbu5DfdeO8T+o7jFgNzmTFrx0.
ECDSA key fingerprint is MD5:db:17:5d:b2:7c:90:19:9f:2a:15:dc:e4:4f:56:f2:4a.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '63.34.46.177' (ECDSA) to the list of known hosts.
Last login: Mon Jul  8 15:32:58 2019

       __|  __|_  )
       _|  (     /   Amazon Linux AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-ami/2018.03-release-notes/
4 package(s) needed for security, out of 4 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-172-31-25-72 ~]$
```

You could also have checked the cluster status to get the connection details:
```bash
$ pcluster status apjc1
Status: CREATE_COMPLETE
MasterServer: RUNNING
MasterPublicIP: 63.34.46.177
ClusterUser: ec2-user
MasterPrivateIP: 172.31.25.72
```

Now that you are connected on the management node, you'll be able to run your first SLURM commands.

First, we'll look at the compute pool using ```sinfo```:
```bash
$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
compute*     up   infinite      2   idle ip-172-31-17-0,ip-172-31-31-55
```

Then we can check whether there are jobs waiting in the queue using ```squeue```:
```bash
$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
$
```
As expected, at this stage the queue is empty.

SLURM allows to run command interactively using ```srun```, so let's test that:
```bash
$ srun -n 1 /bin/hostname
ip-172-31-17-0
```

We could also show the details of a node using ```scontrol```:
```bash
$ scontrol show node ip-172-31-31-55
NodeName=ip-172-31-31-55 Arch=x86_64 CoresPerSocket=1
   CPUAlloc=0 CPUTot=8 CPULoad=0.00
   AvailableFeatures=(null)
   ActiveFeatures=(null)
   Gres=(null)
   NodeAddr=ip-172-31-31-55 NodeHostName=ip-172-31-31-55 Version=18.08
   OS=Linux 4.14.121-85.96.amzn1.x86_64 #1 SMP Wed May 22 00:45:50 UTC 2019
   RealMemory=31144 AllocMem=0 FreeMem=30289 Sockets=8 Boards=1
   State=IDLE ThreadsPerCore=1 TmpDisk=17068 Weight=1 Owner=N/A MCS_label=N/A
   Partitions=compute
   BootTime=2019-07-08T15:33:57 SlurmdStartTime=2019-07-08T15:36:58
   CfgTRES=cpu=8,mem=31144M,billing=8
   AllocTRES=
   CapWatts=n/a
   CurrentWatts=0 LowestJoules=0 ConsumedJoules=0
   ExtSensorsJoules=n/s ExtSensorsWatts=0 ExtSensorsTemp=n/s
```

Ok, now we have checked that everything works the way we expected it to work, so now we can do something more interesting.

Using your favorite text editor, let's write (well, copy-paste) a small MPI code, in a file code ```pi-mpi.c```:
```C
#include <stdio.h>
#include <math.h>
#include <mpi.h>
#include <unistd.h>
#define pi 3.14159265358979323846

int main(int argc, char *argv[])
{
  double total, h, sum, x;
  double t1, t2;
  long long int i, n = 1e10;
  int rank, numprocs;
  char hostname[256];

  MPI_Init(&argc, &argv);
  MPI_Comm_rank(MPI_COMM_WORLD, &rank);
  MPI_Comm_size(MPI_COMM_WORLD, &numprocs);
  t1 = MPI_Wtime();

  h = 1./n;
  sum = 0.;

  gethostname(hostname,255);
  printf("Process number: %d out of %d on host %s\n", rank, numprocs, hostname);

  for (i = rank+1; i <= n; i += numprocs) {
    x = h * ( i - 0.5 );
    sum += 4.0 / ( 1.0 + pow(x,2));
  }

  sum *= h;
  MPI_Reduce(&sum,&total,1,MPI_DOUBLE,MPI_SUM,0,MPI_COMM_WORLD);

  if (rank == 0) {
    t2 = MPI_Wtime();
    printf("Elapsed time is %f\n", t2 - t1 );
    printf("%.17g  %.17g\n", total, fabs(total-pi));
  }

  MPI_Finalize();
  return 0;
}
```

To compile this code, GCC and openmpi have been pre-installed by AWS ParallelCluster, so we just need to setup the MPI environment:
```bash
$ module load openmpi/3.1.4
$ which mpicc
/opt/amazon/efa/bin/mpicc
```

Now that everything is setup the right way, we can compile the application:
```bash
$ mpicc -lm -o pi-mpi pi-mpi.c
$ ls -l
total 16
-rwxrwxr-x 1 ec2-user ec2-user 9392 Jul  8 16:11 pi-mpi
-rw-rw-r-- 1 ec2-user ec2-user  894 Jul  8 16:01 pi-mpi.c
$ ldd pi-mpi
        linux-vdso.so.1 =>  (0x00007ffd9ff86000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f52f7665000)
        libmpi.so.40 => /opt/amazon/efa/lib64/libmpi.so.40 (0x00007f52f736a000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f52f714e000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f52f6d81000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f52f7967000)
        libopen-rte.so.40 => /opt/amazon/efa/lib64/libopen-rte.so.40 (0x00007f52f6acd000)
        libopen-pal.so.40 => /opt/amazon/efa/lib64/libopen-pal.so.40 (0x00007f52f67c7000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f52f65c3000)
        libudev.so.0 => /lib64/libudev.so.0 (0x00007f52f63b3000)
        librt.so.1 => /lib64/librt.so.1 (0x00007f52f61ab000)
        libutil.so.1 => /lib64/libutil.so.1 (0x00007f52f5fa8000)
        libz.so.1 => /lib64/libz.so.1 (0x00007f52f5d92000)
```

So we now have a MPI-enabled binary that is linked with the openmpi library.

Again, using your preferred text editor (vi or nano) create a batch.sh file. It will allow us to submit the ```pi-mpi``` executable as a batch job:
```bash
#!/bin/sh
mpirun ./pi-mpi
```

And we can now lauch that script as a batch job, let's say on 4 cores:
```bash
$ sbatch -n 4 ./batch.sh
```

This time ```squeue``` and ```sinfo``` will return different results:
```bash
$ sbatch -n 4 ./batch.sh
Submitted batch job 6
$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
compute*     up   infinite      1    mix ip-172-31-17-0
compute*     up   infinite      1   idle ip-172-31-31-55
$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
                 6   compute batch.sh ec2-user  R       0:06      1 ip-172-31-17-0
```

Now you can play with the cluster, submitting more jobs of different sizes and see how the cluster will grow an shrink depending on the queue depth.

Once you've played enough with the cluster, you can delete it, which will tear down the CloudFormation stack through which the cluster has been created:
```bash
$ pcluster delete apjc1
```

## Part 4 - Enhanced cluster with EFA and Lustre

### Regional availability

Regions with C5n:

  * ap-northeast-1 (NRT55/NRT7)
  * us-east-1 (all but IAD5/55)
  * us-east-2
  * us-west-2
  * eu-west-1 
  * eu-west-2 (FRA53/FRA54)

FSx for Lustre is available in all the regions where C5n is available (don't take it as a rule, it's not).

### Deployment

We'll start from the cluster definition we used at part1, which is still in ```~/.parallelcluster/config```, but we'll add/change a few parameters:
  * We'll change compute nodes to c5n.18xl
  * We'll add a section for EFA
  * We'll set a lower max size so that it can fit with your account limits
  * We'll add a FSx section

So open the ```~/.parallelcluster/config``` with your preferred editor, and modify it so it would look like the following one:
```bash
[aws]
aws_region_name = <your_preferred_region>

[cluster default]
key_name = <your_key>
vpc_settings = public
scheduler = slurm
master_instance_type = t3.xlarge
compute_instance_type = c5n.18xlarge  <--- Compute instance type changed to c5n.18xlarge
initial_queue_size = 2
max_queue_size = 4                  <--- Max number of compute nodes
maintain_initial_size = true
cluster_type = ondemand
extra_json = { "cluster" : { "cfn_scheduler_slots" : "cores"} }
base_os = alinux
enable_efa = compute                <--- Enables EFA
placement_group = DYNAMIC           <--- Set a cluster placement group for EFA

[fsx fs]
shared_dir = /fsx
storage_capacity = 3600

[vpc public]
vpc_id = vpc-xxxxxxx
master_subnet_id = subnet-xxxxxxx

[global]
cluster_template = default
update_check = true
sanity_check = true

[aliases]
ssh = ssh {CFN_USER}@{MASTER_IP} {ARGS}
```

And you can deploy it like you did previously:
```bash
$ pcluster create apjc2
Beginning cluster creation for cluster: apjc2
Creating stack named: parallelcluster-apjc2
Status: ComputeFleet - CREATE_IN_PROGRESS
Status: parallelcluster-apjc2 - CREATE_COMPLETE
MasterPublicIP: 18.200.40.27
ClusterUser: ec2-user
MasterPrivateIP: 172.31.30.190
```
And again, it will take a while to be created, so let's take some time to dig on EFA.

## Part 5 - EFA explanations

Let's have a look to the EFA presentation.

## Part 6 - Play with EFA

Now that our EFA cluster is ready, we'll use to run your (maybe) first LINPACK.

```bash
$ pcluster ssh apjc2
```

The LINPACK is the code that is used to benchmark the largest supercomputers and rank them in the top500 (https://www.top500.org/lists/).

So let's first get the LINPACK source code:
```bash
$ wget http://www.netlib.org/benchmark/hpl/hpl-2.3.tar.gz
```

```bash
$ tar xvzf hpl-2.3.tar.gz
$ cd hpl-2.3/
$ wget https://raw.githubusercontent.com/arthurpetitpierre/apjc-pcluster-workshop/master/Make.efa
$ module load openmpi/3.1.4
$ make arch=efa
$ cp bin/efa/xhpl ~/
```

Now you have a ```xhpl```binary at the root of your home.

Let's get a configuration file for it:
```bash
$ cd ~/
$ wget https://raw.githubusercontent.com/arthurpetitpierre/apjc-pcluster-workshop/master/HPL.dat
```

Then let's prepare a batch (name it ```batch.sh```) file for the LINPACK run:
```bash
#!/bin/sh
mpirun ./xhpl
```

And then we can run the linpack on two full c5n.18xl:
```bash
$ sbatch -n 72 ./batch.sh
```

Now you're left with the last exercise: try to build the biggest possible cluster, run a linpack, and submit it to the top500.

That's exactly what one of our customers did: https://medium.com/descarteslabs-team/thunder-from-the-cloud-40-000-cores-running-in-concert-on-aws-bf1610679978


## Part 7 - Conclusion

During this workshop, you've learnt to:
  * Build a cluster
  * Compile and run a MPI application
  * Build a cluster with EFA and FSx for Lustre
  * Run a linpack
