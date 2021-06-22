# <img src="https://github.com/oci-hpc/oci-hpc-runbook-lsdyna/blob/master/lsdyna_logo.png" height="60"> LS-DYNA Runbook

# Introduction
This Runbook will take you through the process of deploying an LS Dyna cluster on Oracle Cloud with low latency networking between the compute nodes. Running LS Dyna on Oracle Cloud is quite straightforward, follow along this guide for all the tips and tricks.

LS-DYNA is a general-purpose finite element program capable of simulating complex real world problems. It is used by the automobile, aerospace, construction, military, manufacturing, and bioengineering industries. [LS-DYNA Website](https://www.lstc.com/products/ls-dyna)

<p align="center">
<img src="https://github.com/oci-hpc/oci-hpc-runbook-lsdyna/blob/master/3cars.jpg" height="200" >
 </p>

## Prerequisites

- Permission to `manage` the following types of resources in your Oracle Cloud Infrastructure tenancy: `vcns`, `internet-gateways`, `route-tables`, `network-security-groups`, `subnets`, and `instances`.

- Quota to create the following resources: 1 VCN, 2 subnets, 1 Internet Gateway, 1 NAT Gateway, 1 Service Gateway, 3 route rules, and minimum 2 compute instances in instance pool or cluster network (plus bastion host).

If you don't have the required permissions and quota, contact your tenancy administrator. See [Policy Reference](https://docs.cloud.oracle.com/en-us/iaas/Content/Identity/Reference/policyreference.htm), [Service Limits](https://docs.cloud.oracle.com/en-us/iaas/Content/General/Concepts/servicelimits.htm), [Compartment Quotas](https://docs.cloud.oracle.com/iaas/Content/General/Concepts/resourcequotas.htm).

## Deploy Using Oracle Resource Manager

1. Click [![Deploy to Oracle Cloud](https://oci-resourcemanager-plugin.plugins.oci.oraclecloud.com/latest/deploy-to-oracle-cloud.svg)](https://cloud.oracle.com/resourcemanager/stacks/create?region=home&zipUrl=https://github.com/oracle-quickstart/oci-hpc-runbook-openfoam/releases/latest/download/oci-hpc-runbook-lsdyna-stack-latest.zip)

    If you aren't already signed in, when prompted, enter the tenancy and user credentials.

2. Review and accept the terms and conditions.

3. Select the region where you want to deploy the stack.

4. Follow the on-screen prompts and instructions to create the stack.

5. After creating the stack, click **Terraform Actions**, and select **Plan**.

6. Wait for the job to be completed, and review the plan.

    To make any changes, return to the Stack Details page, click **Edit Stack**, and make the required changes. Then, run the **Plan** action again.

7. If no further changes are necessary, return to the Stack Details page, click **Terraform Actions**, and select **Apply**. 

## Deploy Using the Terraform CLI

### Clone the Module
Now, you'll want a local copy of this repo. You can make that with the commands:

    git clone https://github.com/oracle-quickstart/oci-hpc-runbook-lsdyna.git
    cd oci-hpc-runbook-fluent
    ls

### Set Up and Configure Terraform

1. Complete the prerequisites described [here](https://github.com/cloud-partners/oci-prerequisites).

2. Create a `terraform.tfvars` file, and specify the following variables:

```
# Authentication
tenancy_ocid         = "<tenancy_ocid>"
user_ocid            = "<user_ocid>"
fingerprint          = "<finger_print>"
private_key_path     = "<pem_private_key_path>"

# database
ATP_password           = "<ATP_user_password>"
ATP_data_guard_enabled = false # set the value to true only when you want to enable standby and then re-run terraform apply

# Region
region = "<oci_region>"

# Availablity Domain 
ad = "<availablity doman>" # for example "GrCH:US-ASHBURN-AD-1"

# Bastion 
bastion_ad               = "<availablity doman>" # for example "GrCH:US-ASHBURN-AD-1"
bastion_boot_volume_size = "<bastion_boot_volume_size>" # for example 50
bastion_shape            = "<bastion_shape>" # for example "VM.Standard.E3.Flex"
boot_volume_size         = "<boot_volume_size>" # for example 100
node_count               = "<node_count>" # for example 2
ssh_key                  = "<ssh_key>"
targetCompartment        = "<targetCompartment>" 
use_custom_name          = false
use_existing_vcn         = false
use_marketplace_image    = true
use_standard_image       = true
cluster_network          = false
instance_pool_shape      = "<instance_pool_shape>" # for example VM.Standard.E3.Flex

````

### Create the Resources
Run the following commands:

    terraform init
    terraform plan
    terraform apply

### Destroy the Deployment
When you no longer need the deployment, you can run this command to destroy the resources:

    terraform destroy
    
# Architecture
![](https://github.com/oracle-quickstart/oci-hpc-runbook-fluent/blob/main/images/architecture-hpc.png "Architecture for Running StarCCM+ in OCI")
The architecture for this runbook is as follow, we have one small machine (bastion) that you will connect into. The compute nodes will be on a separate private network linked with RDMA RoCE v2 networking. The bastion will be accesible through SSH from anyone with the key (or VNC if you decide to enable it). Compute nodes will only be accessible through the bastion inside the network. This is made possible with 1 Virtual Cloud Network with 2 subnets, one public and one private.

The above baseline infrastructure provides the following specifications:
-	Networking
    -	1 x 100 Gbps RDMA over converged ethernet (ROCE) v2
    -	Latency as low as 1.5 µs
-	HPC Compute Nodes (BM.HPC2.36)
    -	6.4 TB Local NVME SSD storage per node
    -	36 cores per node
    -	384 GB memory per node

## Install MPI librairies
### Intel MPI 2018

Run those commands on every node. 
```
cd /nfs/cluster
wget https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS-2019.PUB
sudo rpm --import GPG-PUB-KEY-INTEL-SW-PRODUCTS-2019.PUB
sudo yum-config-manager --add-repo=https://yum.repos.intel.com/mpi
sudo yum install -y intel-mpi-2018.4-057 intel-mpi-samples-2018.4-274
```
	
# Running LS-DYNA
Running LS-DYNA is pretty straightforward.
To specify the host you need to run on, you need to create a machinefile. You can generate it as follow, or manually. Format is hostname:corenumber for both Platform and IntelMPI.
```
sed 's/$/:36/' /etc/opt/oci-hpc/hostfile > machinefile
```
Some run parameters can be specified by a parameter file: `pfile`

```
gen { nodump nobeamout dboutonly }
dir { global /mnt/nfs-share/benchmark/one_global_dir local /dev/shm }
```

This particular pfile tells LSDyna not to dump to much information to the disk. Uses memory to store local files and store global files into `/mnt/nfs-share/benchmark/one_global_dir`.

ANother place to store local files if it does not fit in the memory is `/mnt/localdisk/tmp` to use the local NVMe on the machine to store those files.

To run on multiple nodes, place the model on the share drive (Ex:/mnt/nfs-share/work/).
Example provided here is to run the 3 cars model. . You can add it to object storage like the installer and download it or scp it to the machine.
```
wget https://objectstorage.us-phoenix-1.oraclecloud.com/p/qwbdhqwdhqh/n/tenancy/b/bucket/o/3cars_shell2_150ms.k
```
Make sure you have set all the right variables for mpi to run correctly. 
Run it with the following command for Intel MPI (change the modelname and core number):
```
mpirun -np 256 -hostfile ./hostfile_rank \
-ppn 32 -iface enp94s0f0 -genv I_MPI_FABRICS=shm:dapl -genv DAT_OVERRIDE=/etc/dat.conf -genv I_MPI_DAT_LIBRARY=/usr/lib64/libdat2.so \
-genv I_MPI_DAPL_PROVIDER=ofa-v2-cma-roe-enp94s0f0 -genv I_MPI_FALLBACK=0 -genv I_MPI_DAPL_UD=0 -genv I_MPI_ADJUST_ALLREDUCE 5 -genv I_MPI_ADJUST_BCAST 1 -genv I_MPI_DEBUG=4 \
-genv I_MPI_PIN_PROCESSOR_LIST=0-35 -genv I_MPI_PROCESSOR_EXCLUDE_LIST=36-71 \
/mnt/nfs-share/LSDYNA/ ls-dyna_mpp_s_r9_2_119543_x64_redhat54_ifort131_sse2_intelmpi-413 
i=3cars_shell2_150ms.k \
memory=1000m memory2=160m p=pfile
```
 
For platform MPI: 
```
mpirun -np 256 -hostfile ./hostfile_rank -ppn 32 \ 
-d -v -prot -intra=shm -e MPI_FLAGS=y -e MPI_HASIC_UDAPL=ofa-v2-cma-roe-enp94s0f0 -UDAPL \
/mnt/nfs-share/LSDYNA/ls-dyna_mpp_s_r9_2_119543_x64_redhat54_ifort131_sse2_platformmpi \
i=3cars_shell2_150ms.k \
memory=1000m memory2=160m p=pfile
```

## Special case

For some model, Douple Precision executables will not be enough to do the decomposition of the model. You can either use the double precision version of the executable or you can do the decomposition in double precision and still do the run in single precision to gain speed. 
			
For the decomposition, add `ncycles=2` to the command line and add this part to the pfile: 	
```
decomposition {								
 file decomposition								
 rcblog rcblog								
}
```
The model will be decomposed in a file called decomposition and stored in the directory of the model. During the second run, LS-Dyna will see this file and not redo the decomposition. 
								
For the run in Single precision, you can use the same commands in the pfile.
