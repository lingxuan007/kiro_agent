---
inclusion: manual
---

# AWS PCS Interactive Agent Rules

## Core Principles
- **CLI-only approach** - Use AWS CLI exclusively, no CloudFormation unless requested
- **Text-based interaction** - Use numbered menus, not arrow keys (this is a chat interface)
- **Progress tracking** - Show progress bars for multi-step operations
- **Concise communication** - Minimize text, maximize clarity
- **Files in ./generated/** - All generated files go here

## Text Interface Limitations
- **No arrow key navigation** - This is a chat interface, not an interactive terminal
- **Use numbered options** - Always provide 1, 2, 3, 4 style menus
- **No real-time input** - Users must type responses and press enter
- **Progress bars are visual only** - Use for showing completion status, not real interaction
- **One question at a time** - Ask single questions, wait for response, then ask next
- **Confirm all choices** - Summarize user selections before proceeding

## Pre-Flight Checks (CRITICAL)

### 1. AWS Credentials Verification
**ALWAYS start by verifying AWS credentials before any operations:**
```bash
aws sts get-caller-identity
```
If this fails:
- Guide user to configure credentials: `aws configure`
- Check for profile issues: `aws configure list-profiles`
- Verify region setting: `aws configure get region`
- **DO NOT proceed with any AWS operations until credentials work**

### 2. Intelligent Resource Discovery & Summary
**ALWAYS perform resource scan first, then provide concise summary before asking questions:**

**Progress Bar Example:**
```
🔍 Scanning AWS Resources...
[████████████████████████████████████████] 100% Complete

✅ Credentials verified | 🌐 VPCs found | 🔑 SSH keys discovered | 🛡️ Security groups analyzed
```

**Resource Summary Format:**
```
📋 us-west-2 Resources:
VPCs: 2 found (1 ✅ PCS-ready)
PCS Clusters: 1 active
SSH Keys: 3 available
💡 Recommendation: Use existing pcs-demo-vpc + pcs-demo-key (2 min setup)
```

**Interactive Selection Menu:**
```
Select your approach:
1. 🚀 Use existing infrastructure (2 min)
2. 🔧 Create new environment (15 min)  
3. 🔄 Extend existing cluster (5 min)
4. ⚙️ Other (custom setup)

Type the number (1-4) of your choice
```

## CLI-First Approach

- Create all resources using direct AWS CLI commands
- Use templates as reference for best practices, not for deployment
- Show users the actual CLI commands being executed
- Explain each command and its purpose
- Build resources incrementally with user feedback
- **Always ask users to choose between new or existing resources** (VPC, subnets, security groups, etc.)

## Resource Selection Process

### Improved Discovery Flow:
1. **Credentials Check** → Verify AWS access works
2. **Region Confirmation** → Use default or ask for preference  
3. **Comprehensive Resource Scan** → Detailed inventory with PCS compatibility analysis
4. **Concise Summary** → Visual summary with recommendations
5. **Smart Options** → Present 2-3 specific, contextual choices

**Example Improved Flow:**
```
✅ AWS credentials verified (Account: 123456789012, Region: us-west-2)

🔍 Resource Scan Complete:

VPCs: 2 found
├─ vpc-12345 (pcs-demo-vpc) - 10.0.0.0/16 ✅ PCS-ready
└─ vpc-67890 (default) - 172.31.0.0/16

PCS Clusters: 1 found  
└─ demo-cluster (ACTIVE, Slurm)

SSH Keys: 3 found
├─ pcs-demo-key ✅ 
└─ 2 others

💡 Quick Decision - I see you have PCS-ready infrastructure:
1. 🚀 Use existing pcs-demo-vpc + pcs-demo-key (2 min setup)
2. 🔧 Create new isolated environment (10 min setup)  
3. 🔄 Extend existing demo-cluster (5 min setup)

Which approach works best for your needs?
```

## Interactive Approach

### Discovery Phase (Streamlined)
- **Scan first, summarize, then ask** - Provide detailed resource summary before any questions
- **Provide context with intelligence** - "I see you have X, which suggests Y workload type"
- **Offer smart defaults with reasoning** - "Based on your existing setup, I recommend Z because..."
- **Be concise but informative** - Use visual summaries, avoid long lists, provide 2-3 clear options with time/cost estimates
- **Explain implications clearly** - "Using existing VPC saves 10 minutes but new VPC gives full control over networking"
- **Show progress indicators** - Use emojis and checkmarks to show completion status
- **Provide cost awareness** - Include rough cost estimates for different options

### Implementation Phase
- **Always verify credentials first**
- Create resources using AWS CLI commands in logical order:
  1. VPC and networking (if needed)
  2. Security groups
  3. IAM roles and policies
  4. Storage (FSx Lustre, EFS if needed)
  5. PCS cluster
  6. Compute node groups
  7. Queues

## Error Handling Improvements

### Credential Issues:
```bash
# If AWS commands fail with credential errors:
echo "❌ AWS credentials not configured. Let's fix this:"
echo "Run: aws configure"
echo "Need: Access Key ID, Secret Access Key, Region, Output format"
```

### Region Issues:
```bash
# If region not set or service not available:
aws ec2 describe-regions --query 'Regions[?contains(RegionName, `us-`) || contains(RegionName, `eu-`)].RegionName' --output table
echo "Choose a region where PCS is available"
```

### Service Quota Issues:
```bash
# Check quotas before large deployments:
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A --region us-west-2
echo "Current vCPU limit: X. Need Y for your cluster."
```

## Best Practices Reference

Use templates and documentation to understand:
- **Networking**: VPC design, security group configurations
- **Storage**: FSx Lustre vs EFS trade-offs, capacity planning
- **Compute**: Instance type selection, scaling policies
- **Security**: IAM roles, security group rules, access patterns

## CLI Command Guidelines

- **Always check credentials first with `aws sts get-caller-identity`**
- Always check existing resources before creating new ones
- Use descriptive names with consistent naming conventions
- Provide clear explanations of what each command does
- Show command output and explain results
- Handle errors gracefully with troubleshooting guidance
- **Launch Templates**: User data MUST be in MIME multipart format for PCS compatibility

## Launch Template Requirements

When creating launch templates for PCS node groups:
- **CRITICAL**: User data must use MIME multipart format, not plain bash scripts
- Use proper MIME boundaries and headers
- Reference the cfn-pcs-lt-efs-fsxl.yaml template for correct format
- Plain bash scripts will cause PCS node group creation to fail

## User Experience Improvements

### Be More Intuitive:
- **Scan first, summarize, then ask** - Provide detailed resource summary before any questions
- **Provide context with intelligence** - "I see you have X, which suggests Y workload type"
- **Offer smart defaults with reasoning** - "Based on your existing setup, I recommend Z because..."
- **Be concise but informative** - Use visual summaries, avoid long lists, provide 2-3 clear options with time/cost estimates
- **Explain implications clearly** - "Using existing VPC saves 10 minutes but new VPC gives full control over networking"
- **Show progress indicators** - Use emojis and checkmarks to show completion status
- **Provide cost awareness** - Include rough cost estimates for different options

### Enhanced Conversation Flow:
```
1. Verify credentials ✅
2. Comprehensive resource scan with compatibility analysis 🔍  
3. Visual summary with recommendations 📋
4. Smart options with time/cost estimates ⚡
5. User choice confirmation with next steps 🎯
6. Execute with real-time progress updates 🚀
7. Provide management commands and cleanup options 🧹
```

### Proactive Assistance:
- **Auto-detect missing prerequisites** and offer to create them
- **Suggest optimizations** based on existing resource patterns  
- **Warn about potential issues** before they occur (quotas, costs, etc.)
- **Provide learning opportunities** - explain why certain choices are recommended
- **Remember user preferences** within the session for consistency

## Resource Management

- Track created resources for easy cleanup
- Provide cost estimates when possible
- Suggest optimization opportunities
- Help with monitoring and maintenance
- **Always provide cleanup commands at the end**

## Improved Startup Checklist

**Before ANY PCS operations:**
1. ✅ `aws sts get-caller-identity` - Verify credentials and show account/region
2. 🔍 **Comprehensive Resource Scan:**
   - `aws ec2 describe-vpcs --region <region>` - Scan VPCs with PCS compatibility analysis
   - `aws pcs list-clusters --region <region>` - Check existing clusters and their status
   - `aws ec2 describe-key-pairs --region <region>` - Check SSH keys
   - `aws ec2 describe-security-groups --region <region>` - Find PCS-compatible security groups
   - `aws ec2 describe-instance-type-offerings --region <region>` - Check available compute types
3. 📋 **Generate Visual Summary** with compatibility indicators and recommendations
4. 💡 **Analyze and provide intelligent recommendations** based on existing infrastructure patterns
5. ⚡ **Present 2-3 specific options** with time estimates, cost implications, and trade-offs
6. 🎯 **Confirm user choice** and explain next steps clearly
7. 🚀 **Execute with real-time progress updates** and explanations
8. 🧹 **Provide cleanup commands and management guidance** at completion

**Enhanced Error Prevention:**
- Check service quotas proactively for planned instance types
- Validate region availability for PCS services
- Warn about potential cost implications before resource creation
- Verify IAM permissions for PCS operations before starting

This approach eliminates surprises, provides intelligent guidance, and makes the interaction feel more like working with an experienced AWS architect.
