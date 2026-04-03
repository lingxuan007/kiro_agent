---
inclusion: always
---

# Technology Stack

## Core Technologies

### AWS Services
- **AWS PCS**: Parallel Computing Service with Slurm scheduler
- **CloudFormation**: Infrastructure as Code for reproducible deployments
- **AWS CLI**: Primary interface for resource management and automation
- **EC2**: Compute instances (c6i.xlarge, m6i.large, r6i.xlarge families)
- **VPC**: Networking with public/private subnet architecture
- **FSx Lustre**: High-performance parallel file system
- **EFS**: Shared NFS storage for home directories
- **IAM**: Identity and access management with least privilege

### Development Tools
- **Kiro CLI**: AI-powered development assistant with custom agents
- **Amazon Q Developer**: Code generation and assistance
- **YAML**: CloudFormation template format
- **JSON**: AWS CLI parameter format
- **Bash**: Scripting and automation
- **Markdown**: Documentation and steering files

### HPC Stack
- **Slurm**: Workload manager and job scheduler (version 25.05)
- **OpenMPI**: Message Passing Interface for parallel computing
- **Intel MPI**: Alternative MPI implementation
- **GNU Compiler Collection**: C/C++/Fortran compilers
- **Intel oneAPI**: Optimized compilers and libraries

## Technical Constraints

### AWS Limits
- **Service Quotas**: vCPU limits vary by instance family and region
- **Regional Availability**: PCS available in limited AWS regions
- **Network Performance**: Instance types determine network bandwidth
- **Storage Limits**: FSx Lustre capacity and throughput constraints

### Security Requirements
- **VPC Isolation**: All resources within dedicated VPC
- **Private Subnets**: Compute nodes isolated from internet
- **Security Groups**: Minimal required ports (SSH, Slurm, NFS)
- **IAM Roles**: Instance profiles with PCS-specific permissions
- **No Public IPs**: Compute nodes access internet via NAT Gateway

### Performance Considerations
- **Instance Store**: Use NVMe SSD for temporary high-performance storage
- **Placement Groups**: Cluster placement for low-latency communication
- **Enhanced Networking**: SR-IOV and DPDK support for HPC workloads
- **CPU Optimization**: Intel Turbo Boost and hyperthreading configuration

## Preferred Libraries and Frameworks

### Infrastructure
- **CloudFormation**: Preferred over Terraform for AWS-native deployments
- **AWS CLI v2**: Latest version with improved performance
- **jq**: JSON processing for CLI output parsing

### Monitoring and Observability
- **CloudWatch**: Metrics, logs, and alarms
- **AWS Systems Manager**: Instance management and patching
- **Cost Explorer**: Resource cost tracking and optimization

## Development Standards

### File Organization
- Templates in `templates/` directory
- Generated files in `generated/` directory
- Documentation in root and `.kiro/steering/`
- Agent configurations in `.kiro/agents/`

### Naming Conventions
- **Resources**: kebab-case with descriptive prefixes (pcs-demo-vpc)
- **Files**: lowercase with hyphens (cluster-config.json)
- **CloudFormation**: PascalCase for logical IDs
- **Parameters**: camelCase for consistency with AWS APIs
