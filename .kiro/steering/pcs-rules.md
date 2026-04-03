---
inclusion: fileMatch
fileMatchPattern: "templates/**/*.yaml"
---

# AWS PCS CloudFormation Template Rules

## General Guidelines

- Always generate CloudFormation templates in `./generated/` directory
- Use timestamp in generated filenames: `pcs-cluster-{timestamp}.yaml`
- Create deployment README.md alongside each template
- Use base template as reference: #[[file:templates/merged-pcs-cluster.yaml]]
- Copy and modify the merged template based on user requirements

## Template Approach

### Base Template
- **merged-pcs-cluster.yaml**: Complete single-stack template with all resources
- Contains VPC, security groups, IAM roles, FSx Lustre, launch templates, PCS cluster
- Proven working template - copy and modify as needed

### Customization Process
1. Copy #[[file:templates/merged-pcs-cluster.yaml]] to `./generated/pcs-cluster-{timestamp}.yaml`
2. Modify parameters, resources, and configurations based on user requests:
   - **Storage**: Adjust FSx capacity, throughput, or add EFS
   - **Compute**: Change instance types, scaling limits, node groups
   - **Networking**: Modify CIDR blocks, subnets, security group rules
   - **PCS**: Update Slurm version, cluster size, queue configuration

## Security Requirements

- Use VPC CIDR blocks instead of security group references to avoid circular dependencies
- Default to FSx Lustre for high-performance storage
- Include proper IAM roles with minimal required permissions
- Use private subnets for compute nodes, public for login nodes

## Template Standards

- Always use YAML format for CloudFormation templates
- Include comprehensive parameter descriptions
- Add resource tags for cost tracking and management
- Provide clear outputs for important resource IDs

## Deployment Process

1. Gather user requirements
2. Copy and customize merged template
3. Create deployment documentation
4. Deploy and monitor stack creation
5. Provide troubleshooting guidance if needed

## Common Pitfalls to Avoid

### DO NOT Create Individual Resources via AWS CLI
- **NEVER** use `aws ec2 create-security-group` or similar direct resource creation
- **NEVER** use `aws iam create-role` for individual resource creation
- **ALWAYS** rely on CloudFormation templates for resource management
- If CloudFormation fails, fix the template - don't bypass it with CLI commands

### Resource Management
- All resources must be defined in CloudFormation templates
- Use CloudFormation for creation, updates, and deletion
- Maintain infrastructure as code principles
- Direct CLI resource creation breaks stack management and causes drift
