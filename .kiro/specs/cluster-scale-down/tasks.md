# 实施计划：集群规模缩减 (cluster-scale-down)

## 概述

基于 `templates/merged-pcs-cluster.yaml` 基础模板，复制并修改关键参数后输出到 `./generated/` 目录。核心变更为将计算节点组 MaxInstanceCount 从 4 改为 2，FSx Lustre 保持 1200 GiB 并添加 PERSISTENT_2 最小值限制注释。同时生成部署说明 README.md。

## 任务

- [x] 1. 创建 generated 目录并生成缩减后的 CloudFormation 模板
  - [x] 1.1 创建 `./generated/` 目录（如不存在）
    - _需求: 5.1_
  - [x] 1.2 复制 `templates/merged-pcs-cluster.yaml` 到 `./generated/pcs-cluster-{timestamp}.yaml`
    - 使用当前时间戳生成文件名，格式如 `pcs-cluster-20250101T120000.yaml`
    - _需求: 5.2, 5.3_
  - [x] 1.3 修改 PCSNodeGroupCompute 的 MaxInstanceCount 从 4 改为 2
    - 修改路径: `Resources.PCSNodeGroupCompute.Properties.ScalingConfiguration.MaxInstanceCount`
    - 确保 MinInstanceCount 保持为 0 不变
    - _需求: 1.1, 1.2_
  - [x] 1.4 更新 FSxCapacity 参数描述并添加 PERSISTENT_2 最小值注释
    - 将 `FSxCapacity` 参数的 Description 更新为: `Storage capacity in GiB (PERSISTENT_2 minimum is 1200 GiB)`
    - 在 FSxCapacity 参数上方添加 YAML 注释说明 PERSISTENT_2 最小容量限制
    - 保持 Default: 1200、MinValue: 1200 不变
    - _需求: 2.1, 2.2, 2.3_
  - [x] 1.5 验证其他配置保持不变
    - 确认 Login 节点组 MinInstanceCount=1, MaxInstanceCount=1 不变
    - 确认集群 Size: SMALL 不变
    - 确认 Slurm 调度器配置不变
    - 确认 demo 队列与 PCSNodeGroupCompute 关联不变
    - 确认 VPC、子网、安全组、IAM 资源配置不变
    - 确认 FSxPerUnitStorageThroughput Default: 125 不变
    - 确认实例类型映射 (c6i.xlarge / c7g.xlarge) 不变
    - _需求: 1.3, 2.4, 3.1, 3.2, 3.3, 4.1, 4.2, 4.3, 4.4, 6.1, 6.2_

- [x] 2. 生成部署说明文档 README.md
  - [x] 2.1 在 `./generated/` 目录下创建 README.md
    - 记录所有变更内容（MaxInstanceCount 4→2）
    - 说明 FSx Lustre 保持 1200 GiB 的原因（PERSISTENT_2 最小值限制）
    - 列出不变的配置项确认清单
    - 提供 CloudFormation 部署命令示例
    - 包含部署前注意事项（如当前运行实例数超过新 MaxInstanceCount 的处理建议）
    - _需求: 5.5_

- [x] 3. 检查点 - 验证生成文件
  - 确认 `./generated/pcs-cluster-{timestamp}.yaml` 文件存在且 YAML 语法有效
  - 确认 `./generated/README.md` 文件存在且内容完整
  - 使用 diff 对比基础模板和生成模板，确认仅有预期变更
  - 确保所有测试通过，如有问题请向用户确认
  - _需求: 5.4_

## 备注

- 所有任务基于 `templates/merged-pcs-cluster.yaml` 基础模板进行修改
- 生成的模板必须保持有效的 CloudFormation YAML 语法
- 本功能为 IaC 模板静态修改，不涉及属性基测试（PBT）
- 检查点任务用于确保增量验证
