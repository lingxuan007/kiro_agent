# AWS PCS Cluster Creation Agent

A specialized Kiro CLI agent for creating and managing AWS Parallel Computing Service (PCS) clusters with Slurm scheduling, based on [AWS HPC Recipes](https://github.com/aws-samples/aws-hpc-recipes) best practices.

## Overview

This agent helps you create production-ready HPC clusters on AWS with:
- **Interactive CLI Approach**: Step-by-step cluster creation using AWS CLI commands
- **Expert Guidance**: Built-in AWS PCS security and performance recommendations
- **Flexible Configuration**: Choose between new or existing AWS resources
- **Best Practices**: Based on AWS HPC Recipes and getting started guides

## Prerequisites

1. **AWS CLI** configured with appropriate permissions
2. **Kiro CLI** with custom agent support
3. **AWS Account** with PCS service available in your region
4. **VPC and Networking** (can be created using provided configurations)

## Quick Start

### 1. Enable Todo Lists (Recommended)
```bash
kiro-cli settings chat.enableTodoList true
```

### 2. Activate the Interactive PCS Agent
```bash
kiro-cli --agent pcs-interactive-specialist
```
Use the interactive agent for step-by-step cluster creation with AWS CLI commands and expert guidance.

### 3. Create Your First Cluster
```
I'm new to AWS PCS. Help me create my first cluster with best practices.
```

### 4. Cleanup Resources
```bash
kiro-cli --agent pcs-cleanup-agent
```
Use the cleanup agent to safely delete any PCS cluster and reset the project to default state.

## Golden Cluster Architecture

Based on the [AWS PCS Getting Started Tutorial](https://docs.aws.amazon.com/pcs/latest/userguide/getting-started.html), the golden cluster includes:

### **Cluster Configuration**
- **Name**: `get-started`
- **Scheduler**: Slurm Version 25.05
- **Controller Size**: Small
- **Networking**: IPv4 with VPC and private subnets

### **Compute Node Groups**
1. **Login Nodes** (`login`)
   - **Instance Type**: c6i.xlarge
   - **Scaling**: Min 1, Max 1 (static)
   - **Subnet**: Public subnet for SSH access
   - **Purpose**: Interactive access and job submission

2. **Compute Nodes** (`compute-1`)
   - **Instance Type**: c6i.xlarge
   - **Scaling**: Min 0, Max 4 (elastic)
   - **Subnet**: Private subnet for security
   - **Purpose**: Job execution with auto-scaling

### **Storage**
- **Amazon EFS**: Shared home directory storage
- **Amazon FSx for Lustre**: High-performance shared storage
- **Local NVMe**: Instance store for temporary data

### **Security**
- **IAM Instance Profile**: `AWSPCS-getstarted-role`
- **Security Groups**: Cluster-specific with minimal required access
- **Network Isolation**: Private subnets for compute nodes
- **AMI**: AWS PCS sample AMI with Slurm pre-configured

## File Structure

```
.
├── README.md                           # This file
├── PCS_CONTEXT.md                      # Detailed PCS knowledge base
├── .kiro/
│   ├── agents/
│   │   ├── pcs-interactive-specialist.json  # Interactive CLI-based agent
│   │   └── pcs-cleanup-agent.json      # Cluster cleanup agent
│   ├── steering/
│   │   ├── product.md                  # Product overview
│   │   ├── tech.md                     # Technology stack
│   │   ├── structure.md                # Project structure
│   │   ├── pcs-rules.md                # CloudFormation template rules
│   │   └── pcs-interactive-rules.md    # Interactive agent rules
│   └── cli-todo-lists/                 # Agent todo list storage
├── templates/
│   └── merged-pcs-cluster.yaml         # Reference CloudFormation template
└── generated/                          # Agent-generated files and configurations
```

## Configuration Format

All configurations use AWS CLI JSON parameter format for compatibility with AWS documentation:

### Cluster Configuration (`cluster-config.json`)
```json
{
  "clusterName": "get-started",
  "scheduler": {
    "type": "SLURM",
    "version": "24.11"
  },
  "size": "SMALL",
  "networking": {
    "subnetIds": ["subnet-XXXXXXXXX"],
    "securityGroupIds": ["sg-XXXXXXXXX"]
  }
}
```

### Node Group Configuration (`login-nodegroup-config.json`)
```json
{
  "computeNodeGroupName": "login",
  "clusterIdentifier": "get-started",
  "instanceConfigurations": [
    {
      "instanceType": "c6i.xlarge"
    }
  ],
  "scalingConfiguration": {
    "minInstanceCount": 1,
    "maxInstanceCount": 1
  },
  "subnetIds": ["subnet-XXXXXXXXX"],
  "iamInstanceProfileArn": "arn:aws:iam::ACCOUNT:instance-profile/AWSPCS-getstarted-role"
}
```

## Common Use Cases

### Creating Your First Cluster
```
I'm new to AWS PCS. Help me create my first cluster with best practices.
```

### Custom Workload Optimization
```
I need a cluster optimized for:
- Computational fluid dynamics simulations
- 500+ core jobs
- GPU acceleration
- High-memory instances
```

### Multi-Region Setup
```
Help me set up PCS clusters in multiple regions with shared storage
```

### Cost Optimization
```
Configure a cost-optimized cluster with spot instances and auto-scaling
```

## Configuration Options

The agent will ask about:

1. **Region Selection**: Where to deploy your cluster
2. **Instance Types**: Compute requirements and budget
3. **Storage Needs**: Shared storage size and performance
4. **Network Setup**: New VPC or existing infrastructure
5. **Security Requirements**: Access patterns and compliance needs
6. **Scaling Policies**: Auto-scaling behavior and limits

## Default Configurations

If you're unsure about specific settings, the agent provides sensible defaults:

- **Small Development**: 1-10 nodes, basic storage, cost-optimized
- **Medium Production**: 10-50 nodes, high-performance storage, balanced
- **Large Scale**: 50+ nodes, parallel file systems, performance-optimized

## Troubleshooting

The agent includes built-in troubleshooting for common issues:
- VPC and subnet configuration problems
- Security group connectivity issues
- IAM permission errors
- Storage mounting failures
- Compute node provisioning delays

## Getting Help

```
# Check cluster status
Show me the status of all my PCS clusters

# Troubleshoot connectivity
My compute nodes can't reach the internet, help diagnose

# Scale cluster
Help me add more compute capacity to my existing cluster

# Monitor costs
Show me the cost breakdown of my PCS resources
```

## Next Steps

1. **Start with Golden Cluster**: Deploy the default configuration first
2. **Learn Slurm**: Familiarize yourself with job submission and management
3. **Optimize for Workload**: Customize instance types and scaling policies
4. **Monitor and Scale**: Use CloudWatch metrics to optimize performance
5. **Implement CI/CD**: Automate cluster deployments with the provided templates

For detailed technical information and best practices, see [PCS_CONTEXT.md](./PCS_CONTEXT.md).