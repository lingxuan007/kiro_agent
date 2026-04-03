# 需求文档

## 简介

本功能旨在将现有 AWS PCS 集群的规模缩减至当前配置的一半，以降低运营成本并优化资源利用率。缩减范围涵盖计算节点组的最大实例数量和 FSx Lustre 存储容量，同时保持集群的基本功能和可用性不受影响。

## 术语表

- **CloudFormation_Template**: AWS CloudFormation YAML 模板文件，定义集群所有基础设施资源
- **Compute_Node_Group**: PCS 集群中用于执行计算任务的 EC2 实例组（当前名称为 compute-1）
- **Login_Node_Group**: PCS 集群中用于用户交互访问和作业提交的 EC2 实例组（当前名称为 login）
- **FSx_Lustre_FileSystem**: Amazon FSx for Lustre 高性能并行文件系统，用于集群共享存储
- **MaxInstanceCount**: 计算节点组允许的最大 EC2 实例数量
- **StorageCapacity**: FSx Lustre 文件系统的存储容量（单位：GiB）
- **PerUnitStorageThroughput**: FSx Lustre 每 TiB 存储的吞吐量（单位：MB/s/TiB）
- **ScaleDown_Template**: 缩减后生成的新 CloudFormation 模板文件

## 需求

### 需求 1：缩减计算节点组最大实例数

**用户故事：** 作为 HPC 集群管理员，我希望将计算节点组的最大实例数减半，以降低计算资源的峰值成本。

#### 验收标准

1. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 将 Compute_Node_Group 的 MaxInstanceCount 从 4 更新为 2
2. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持 Compute_Node_Group 的 MinInstanceCount 为 0 不变
3. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持 Compute_Node_Group 的实例类型 c6i.xlarge 不变

### 需求 2：缩减 FSx Lustre 存储容量

**用户故事：** 作为 HPC 集群管理员，我希望将 FSx Lustre 存储容量减半，以降低存储成本。

#### 验收标准

1. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 将 FSx_Lustre_FileSystem 的 StorageCapacity 默认值从 1200 GiB 更新为 1200 GiB
2. THE CloudFormation_Template SHALL 将 FSxCapacity 参数的 MinValue 保持为 1200
3. IF FSx Lustre PERSISTENT_2 部署类型的最小容量限制为 1200 GiB，THEN THE CloudFormation_Template SHALL 保持 StorageCapacity 为 1200 GiB 并在模板注释中说明无法进一步缩减的原因
4. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持 PerUnitStorageThroughput 为 125 MB/s/TiB 不变

### 需求 3：保持 Login 节点组配置不变

**用户故事：** 作为 HPC 集群管理员，我希望 Login 节点组保持当前最小配置，以确保用户始终能够访问集群。

#### 验收标准

1. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持 Login_Node_Group 的 MinInstanceCount 为 1 不变
2. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持 Login_Node_Group 的 MaxInstanceCount 为 1 不变
3. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持 Login_Node_Group 的实例类型 c6i.xlarge 不变

### 需求 4：保持集群核心配置不变

**用户故事：** 作为 HPC 集群管理员，我希望集群的核心配置（控制器大小、调度器、网络、队列）在缩减过程中保持不变，以确保集群功能的连续性。

#### 验收标准

1. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持 PCS 集群的 Size 为 SMALL 不变
2. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持 Slurm 调度器配置不变
3. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持 demo 队列与 Compute_Node_Group 的关联关系不变
4. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 保持所有 VPC、子网、安全组和 IAM 资源配置不变

### 需求 5：生成符合规范的缩减模板

**用户故事：** 作为 HPC 集群管理员，我希望缩减操作通过生成新的 CloudFormation 模板来实现，以遵循基础设施即代码的最佳实践。

#### 验收标准

1. THE CloudFormation_Template SHALL 将 ScaleDown_Template 生成到 `./generated/` 目录中
2. THE CloudFormation_Template SHALL 使用带时间戳的文件名格式 `pcs-cluster-{timestamp}.yaml`
3. THE ScaleDown_Template SHALL 基于 `templates/merged-pcs-cluster.yaml` 基础模板进行修改
4. THE ScaleDown_Template SHALL 保持有效的 CloudFormation YAML 语法
5. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 同时生成部署说明文档 README.md，记录所有变更内容

### 需求 6：模板参数验证

**用户故事：** 作为 HPC 集群管理员，我希望缩减后的模板参数约束正确反映新的配置范围，以防止部署时出现无效配置。

#### 验收标准

1. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 确保 FSxCapacity 参数的 MinValue 和 MaxValue 约束与 FSx Lustre PERSISTENT_2 部署类型的有效范围一致
2. WHEN 生成 ScaleDown_Template 时，THE CloudFormation_Template SHALL 确保所有参数的 AllowedValues 约束保持有效
