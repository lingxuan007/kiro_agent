# AWS PCS Technical Context and Best Practices

Technical reference for AWS Parallel Computing Service (PCS) cluster creation and management.

## Architecture Overview

### Core Components

**AWS PCS Cluster**
- Persistent resource for managing HPC workloads
- Built around Slurm job scheduler
- Controller runs in AWS-managed service account
- Communicates with compute resources in customer account

**Cluster Controller**
- Manages job scheduling and resource allocation
- Sizes: SMALL (up to 32 instances, 256 jobs), MEDIUM (up to 512 instances, 8,192 jobs), LARGE (up to 2,048 instances, 16,384 jobs)
- Cannot be changed after cluster creation
- Automatically handles failover and scaling
- For requirements beyond LARGE, contact AWS

**Compute Node Groups**
- Collections of EC2 instances for job execution
- Auto-scaling based on job queue demand
- Support multiple instance types within same group (must share processor architecture and vCPU count; GPU types must share GPU count)
- Can be dynamically added/removed from cluster

**Queues**
- Slurm partitions for organizing workloads
- Associate one or more compute node groups with each queue
- Users submit jobs to queues, not directly to compute nodes
- Support priority-based scheduling via Slurm custom settings

## Networking Requirements

### VPC Prerequisites

**Subnet Requirements**
- Minimum 1 available IP address per subnet
- Private subnets recommended for security
- Public subnets only if direct internet access needed
- IPv4 or IPv6 (not both simultaneously)

**VPC Configuration**
- DNS resolution and DNS hostnames must be enabled
- Route tables configured for internet access (via NAT or IGW)
- Sufficient IP address space for planned cluster size
- Consider multiple AZs for high availability

### Security Groups

AWS PCS creates network interfaces in your VPC for the cluster's scheduler endpoints. The PCS controller itself runs in an AWS-managed account — you do not create or manage a separate controller security group. You configure security groups for the cluster's elastic network interfaces and for your compute node groups.

**Cluster Security Group (applied to PCS-managed ENIs in your VPC)**
```
Inbound:
- Port 6817-6819 (TCP) from compute node security groups - Slurm communication (6819 needed if accounting enabled)

Outbound:
- All traffic to compute node security groups - Slurm communication
```

**Compute Node Security Group**
```
Inbound:
- Port 6818 (TCP) from cluster security group - Slurm daemon
- Port 22 (TCP) from login node security groups - SSH access
- All traffic from same security group - Inter-node communication

Outbound:
- Port 6817-6819 (TCP) to cluster security group - Slurm controller and accounting communication
- All traffic to 0.0.0.0/0 - Internet access for packages and data
```

**Login Node Security Group**
```
Inbound:
- Port 22 (TCP) from user access ranges - SSH login

Outbound:
- Port 6817-6819 (TCP) to cluster security group - Slurm client to controller and accounting
- All traffic to 0.0.0.0/0 - General internet access
```

## Storage Architecture

### Shared Storage Options

**Amazon EFS (Recommended for General Use)**
- POSIX-compliant NFS file system
- Automatic scaling and high availability
- Performance modes: General Purpose, Max I/O
- Throughput modes: Elastic (default, recommended for spiky workloads), Provisioned, Bursting
- Encryption at rest and in transit
- Cross-AZ replication available

**Amazon FSx for Lustre (High Performance)**
- High-performance parallel file system
- Optimized for compute-intensive workloads
- S3 integration for data lifecycle management
- Scratch and persistent file system types
- Sub-millisecond latencies, high throughput

**Amazon EBS (Node-Local Storage)**
- Block storage for individual compute nodes
- Multiple volume types: gp3, io2, st1, sc1
- Encryption and snapshot capabilities
- Not shared between nodes

**Additional Supported Storage**
- Amazon FSx for NetApp ONTAP
- Amazon FSx for OpenZFS
- Amazon S3 (via Mountpoint for Amazon S3)
- Amazon File Cache
- Self-managed storage resources

### Storage Best Practices

**Mount Point Standards**
- `/home` - User home directories (EFS)
- `/shared` - Shared application data (EFS/FSx)
- `/scratch` - Temporary job data (local NVMe/EBS)
- `/apps` - Software installations (EFS)

**Performance Considerations**
- Use EFS for shared, moderate-performance needs
- Use FSx for Lustre for high-performance, parallel workloads
- Use local NVMe for temporary, high-IOPS requirements
- Consider data locality for large datasets

## Compute Node Groups

### Instance Type Selection

**CPU-Optimized Families**
- `c7i/c7i-flex` - Balanced compute performance
- `m7i/m7i-flex` - General purpose with memory balance
- `r7i/r7iz` - Memory-optimized for large datasets
- `hpc7a/hpc7g` - HPC-optimized with EFA

**GPU-Enabled Families**
- `p4d/p5/p5en/p6-b200/p6-b300` - ML training and HPC simulations
- `g6/g6e` - Graphics workstations and visualization
- `trn1/trn2` - AWS Trainium for ML training
- `inf2` - AWS Inferentia for ML inference

**Storage-Optimized**
- `i4i` - NVMe SSD for high random I/O
- `d3/d3en` - Dense HDD storage for analytics

### Scaling Configuration

**Auto Scaling Parameters**
- Min capacity: 0 for cost optimization (dynamic scaling)
- Max capacity: Based on budget and workload peaks
- Static configuration: set min and max to the same non-zero value
- Dynamic configuration: set min to 0, max to desired ceiling
- PCS does not support mixed static and dynamic instances in the same node group
- Scale-down idle time is a cluster-level setting (`scaleDownIdleTimeInSeconds` on the cluster's `slurmConfiguration`), not a node group setting

**Purchase Options**
- On-Demand Instances
- Spot Instances (requires AWSServiceRoleForEC2Spot service-linked role)
- Capacity Block (EC2 Capacity Blocks for ML — P and TRN families)
- On-Demand Capacity Reservations (ODCRs)

**Launch Template Requirements**
- Must specify a PCS-compatible AMI. A compatible AMI requires two PCS-provided installers to be run: the PCS agent installer and the Slurm installer. AWS provides sample AMIs with these pre-installed, or you can build your own custom AMI by running both installers on a base image.
- Instance profile with required IAM permissions (must include `RegisterComputeNodeGroupInstance` API permission)
- Security group assignments
- Storage configuration
- Subnet configuration

## Queue Management

### Queue Configuration

Queues in AWS PCS associate one or more compute node groups with a named partition. The PCS CreateQueue API accepts a queue name, cluster identifier, and compute node group configurations. Priority and description are Slurm partition-level settings configured through Slurm custom settings, not PCS API fields.

**Creating a queue (CLI)**
```bash
aws pcs create-queue \
  --region us-east-1 \
  --cluster-identifier my-cluster \
  --queue-name batch \
  --compute-node-group-configurations computeNodeGroupId=cng-ExampleId1
```

**Adding Slurm-level settings (priority, etc.)**
Use Slurm custom settings on the queue or compute node group to configure partition-level parameters like Priority, MaxTime, and other scheduling policies. PCS exposes the most popular `slurm.conf` settings, and as of March 2026 also supports `slurmdbd.conf` and cgroup settings — allowing you to configure accounting behavior, data retention, privacy controls, CPU/memory resource isolation, and device access directly through the PCS console, CLI, or SDK.

### Resource Limits

Resource limits in PCS are configured through Slurm's native mechanisms, not through PCS API fields. Use Slurm accounting (sacctmgr) or Slurm custom settings to configure:

**Per-User Limits (via sacctmgr or slurm.conf)**
- MaxJobs: Maximum concurrent jobs per user
- MaxSubmitJobs: Maximum jobs in queue per user
- MaxWall: Maximum job runtime

**Per-Partition Limits (via Slurm custom settings)**
- MaxTime: Maximum job runtime for the partition
- MaxNodes: Maximum nodes per job
- Priority: Partition priority

## Cluster Sizing Guidelines

### Development/Testing
- **Size**: SMALL (up to 32 instances, 256 jobs)
- **Nodes**: 1-10 compute nodes
- **Instance Types**: t3.medium or c7i.large for debugging and script validation
- **Storage**: 100GB EFS General Purpose
- **Use Case**: Code development, job script testing, small validation runs

### Production/Research
- **Size**: MEDIUM (up to 512 instances, 8,192 jobs)
- **Nodes**: 10-100 compute nodes
- **Instance Types**: c7i.xlarge, hpc7a, r7i.xlarge, p5 (GPU)
- **Storage**: 1TB+ EFS, FSx for Lustre for high-performance
- **Use Case**: Production workloads, research simulations

### Large Scale/Commercial
- **Size**: LARGE (up to 2,048 instances, 16,384 jobs)
- **Nodes**: 100+ compute nodes
- **Instance Types**: hpc7a.96xlarge, c7i.48xlarge, p5.48xlarge, p5en (GPU)
- **Storage**: Multi-TB FSx for Lustre, tiered storage
- **Use Case**: Commercial HPC, AI/ML training

## Cost Optimization

### Instance Strategy
- Use Spot Instances for fault-tolerant workloads (up to 90% savings)
- Mix On-Demand and Spot for availability
- Right-size instances based on workload profiling
- Consider Savings Plans or Reserved Instances for steady-state capacity

### Storage Optimization
- EFS Infrequent Access for archive data
- FSx scratch file systems for temporary data
- S3 integration for long-term data storage
- Lifecycle policies for automated tiering

### Operational Efficiency
- Set compute node group min to 0 so nodes scale down when idle (node management fees stop; controller fee continues while cluster exists)
- Use CloudWatch metrics for utilization monitoring
- Set up billing alerts and cost tracking
- Delete unused clusters to stop controller fees

### PCS Service Fees
- Hourly cluster controller fee (varies by controller size: Small/Medium/Large)
- Hourly per-instance node management fee: standard tier for most EC2 types, advanced tier for P and TRN families
- Optional: Slurm accounting usage fee + accounting storage fee
- All PCS fees are in addition to underlying EC2, storage, and networking costs

## IAM Roles and Permissions

### PCS Service-Linked Role
AWS PCS uses a service-linked role for managing resources in your account. This is created automatically.

### Compute Node Instance Profile
**Required Permissions**:
- Permission to call `pcs:RegisterComputeNodeGroupInstance`
- `AmazonSSMManagedInstanceCore` - Systems Manager access
- `CloudWatchAgentServerPolicy` - Monitoring (optional)
- Custom policy for EFS/FSx access as needed

AWS PCS can create a basic instance profile for you during compute node group creation in the console.

### User Access Policies
There are no AWS-published managed policies named `PCSFullAccess` or `PCSReadOnlyAccess`. Create custom IAM policies scoped to the `pcs:*` actions your users need. Example actions:
- `pcs:CreateCluster`, `pcs:DeleteCluster`, `pcs:GetCluster`, `pcs:ListClusters`
- `pcs:CreateComputeNodeGroup`, `pcs:UpdateComputeNodeGroup`
- `pcs:CreateQueue`, `pcs:UpdateQueue`

## Slurm Configuration

### Key Parameters

**SelectTypeParameters**
- `CR_CPU` - CPU-only scheduling
- `CR_CPU_Memory` - Memory-aware scheduling (recommended)

Configure via Slurm custom settings when creating or updating a cluster.

**Accounting Configuration**
- PCS manages the accounting database natively — no external RDS needed
- Enable via `slurmConfiguration.accounting` with `mode=STANDARD`
- Set `defaultPurgeTimeInDays` for record retention (-1 for indefinite, valid range: -1 to 10000, 0 is invalid)
- Requires Slurm version 24.11 or later
- Accounting uses port 6819 (slurmdbd) — ensure security groups allow TCP 6819 between the cluster ENIs and compute nodes
- Enabling accounting incurs additional PCS billing charges (accounting usage fee + storage fee)
- As of March 2026, PCS also supports `slurmdbd.conf` custom settings for fine-tuning accounting behavior (privacy controls, data retention, workload tracking)

**Prolog/Epilog Scripts**
- Must be directories, not files
- Run before/after each job
- Useful for environment setup and cleanup

### Job Submission Examples

**Basic CPU Job**
```bash
#!/bin/bash
#SBATCH --job-name=my-job
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=4
#SBATCH --time=01:00:00
#SBATCH --partition=batch

srun my-application
```

**GPU Job**
```bash
#!/bin/bash
#SBATCH --job-name=gpu-job
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --time=02:00:00
#SBATCH --partition=gpu

nvidia-smi
srun ./gpu-application
```

## Monitoring and Troubleshooting

### CloudWatch Metrics
- Cluster health and job queue status
- Compute node utilization and scaling events
- Storage performance and capacity metrics
- PCS delivers metrics and application logs to CloudWatch
- Auditable events emitted to CloudTrail

### Common Issues

**Compute Nodes Not Starting**
- Check security group rules (ensure Slurm ports 6817-6819 are open between cluster and nodes; 6819 needed for accounting)
- Verify IAM instance profile permissions (RegisterComputeNodeGroupInstance)
- Review launch template configuration
- Check subnet capacity and service quotas

**Jobs Stuck in Queue**
- Verify resource requests vs availability
- Check queue configuration and compute node group associations
- Review node state and availability
- Examine Slurm logs for errors

**Storage Access Issues**
- Verify EFS/FSx mount targets in correct subnets
- Check security group rules for NFS (port 2049)
- Review IAM permissions for storage access
- Test network connectivity to storage endpoints

### Log Locations
- Slurm logs: `/var/log/slurm/`
- System logs: `/var/log/messages`
- CloudWatch Logs for centralized logging
- AWS CloudTrail for API activity

## Integration Patterns

### Infrastructure as Code
- AWS CloudFormation support for PCS clusters and associated infrastructure
- Terraform via AWS provider
- EC2 Image Builder for AMI build automation

### Data Pipeline Integration
- S3 data lakes for input/output
- AWS Batch for containerized workloads
- Step Functions for workflow orchestration
- EventBridge for job completion notifications

### ML/AI Workflows
- SageMaker integration for model training
- Jupyter notebooks for interactive development
- Container-based workloads with Docker/Singularity

### Directory Services
- LDAP-based user authentication and authorization
- Microsoft Active Directory, Microsoft Entra ID, OpenLDAP

## CLI Command Reference

### Cluster Management
```bash
# Create cluster (Slurm 25.05 is current; 24.11 also supported)
aws pcs create-cluster \
  --region us-east-1 \
  --cluster-name my-cluster \
  --scheduler '{"type":"SLURM","version":"25.05"}' \
  --size SMALL \
  --networking '{"subnetIds":["subnet-ExampleId1"],"securityGroupIds":["sg-ExampleId1"]}'

# Create cluster with accounting and custom Slurm settings
aws pcs create-cluster \
  --region us-east-1 \
  --cluster-name my-cluster \
  --scheduler '{"type":"SLURM","version":"25.05"}' \
  --size MEDIUM \
  --networking '{"subnetIds":["subnet-ExampleId1"],"securityGroupIds":["sg-ExampleId1"]}' \
  --slurm-configuration '{"scaleDownIdleTimeInSeconds":3600,"accounting":{"mode":"STANDARD"},"slurmCustomSettings":[{"parameterName":"SelectTypeParameters","parameterValue":"CR_CPU_Memory"}]}'

# List clusters
aws pcs list-clusters --region us-east-1

# Describe cluster (use --cluster-identifier, accepts name or ID)
aws pcs get-cluster --region us-east-1 --cluster-identifier my-cluster

# Delete cluster (all queues and compute node groups must be deleted first)
aws pcs delete-cluster --region us-east-1 --cluster-identifier my-cluster
```

### Compute Node Groups
```bash
# Create compute node group (required params: --cluster-identifier, --compute-node-group-name,
# --subnet-ids, --instance-configs, --custom-launch-template, --iam-instance-profile-arn,
# --scaling-configuration. --ami-id is optional if launch template specifies an AMI)
aws pcs create-compute-node-group \
  --region us-east-1 \
  --cluster-identifier my-cluster \
  --compute-node-group-name batch-nodes \
  --ami-id ami-ExampleId1 \
  --subnet-ids subnet-ExampleId1 \
  --instance-configs '[{"instanceType":"c7i.xlarge"}]' \
  --custom-launch-template id=lt-ExampleId1,version=1 \
  --iam-instance-profile-arn arn:aws:iam::123456789012:instance-profile/my-pcs-profile \
  --scaling-configuration minInstanceCount=0,maxInstanceCount=10

# List compute node groups
aws pcs list-compute-node-groups --region us-east-1 --cluster-identifier my-cluster

# Update scaling configuration
aws pcs update-compute-node-group \
  --region us-east-1 \
  --cluster-identifier my-cluster \
  --compute-node-group-identifier batch-nodes \
  --scaling-configuration maxInstanceCount=20
```

### Queue Management
```bash
# Create queue
aws pcs create-queue \
  --region us-east-1 \
  --cluster-identifier my-cluster \
  --queue-name batch \
  --compute-node-group-configurations computeNodeGroupId=cng-ExampleId1

# List queues
aws pcs list-queues --region us-east-1 --cluster-identifier my-cluster

# Update queue
aws pcs update-queue \
  --region us-east-1 \
  --cluster-identifier my-cluster \
  --queue-identifier batch \
  --compute-node-group-configurations computeNodeGroupId=cng-ExampleId1
```

## Supported Slurm Versions

AWS PCS currently supports Slurm 25.05 and 24.11. Slurm 24.05 has reached end of support. AWS PCS supports up to three major Slurm versions at a time. Once a version reaches end of life, no new clusters can be created with it, but existing clusters continue running for up to 12 months.

## Region Availability

AWS PCS is available in: US East (N. Virginia, Ohio), US West (Oregon), Asia Pacific (Mumbai, Singapore, Sydney, Tokyo), Europe (Frankfurt, Ireland, London, Paris, Stockholm), and AWS GovCloud (US-East, US-West). Region availability is expanding — check the [AWS Regional Services List](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/) for the latest.
