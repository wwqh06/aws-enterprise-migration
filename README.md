# aws-enterprise-migration
AWS Enterprise Web Application Cloud Migration &amp; High-Availability Solution (Multi-AZ, ALB, RDS, S3, IAM, CloudWatch)个人尝试
# AWS 企业网站云上高可用架构迁移与容灾方案
# AWS Enterprise Web Application Cloud Migration & High-Availability Solution

![AWS Region](https://img.shields.io/badge/Region-ap--northeast--1-orange?logo=amazon-aws)
![VPC](https://img.shields.io/badge/VPC-10.0.0.0%2F16-blue)
![HA Architecture](https://img.shields.io/badge/Architecture-Multi--AZ-success)
![Security](https://img.shields.io/badge/Security-Defense--in--Depth-red)
![Zero-Credential](https://img.shields.io/badge/IAM-STS%20Role%20Based-brightgreen)

## 一、 项目背景与业务痛点 (Business Context)

本项目模拟一家中小型电商企业的核心系统上云改造演练。原系统运行于传统 IDC 单台裸金属物理服务器，随着线上访问量增长，面临以下架构瓶颈：
1. **单点故障（SPOF）**：Web 容器（Nginx）与数据库单机混部，缺乏高可用冗余，单点宕机即导致全站瘫痪。
2. **缺乏弹性扩展能力**：无状态应用与有状态持久化数据强绑定，无法应对流量突增时的横向伸缩。
3. **安全边界扁平**：核心数据库与 Web 应用共用公网出入口，缺乏网络层与传输层的隔离防护。
4. **监控与告警滞后**：依赖人工巡检，缺乏端到端指标联动与秒级自动化故障感知机制。

基于 **AWS 架构完善框架（Well-Architected Framework）** 的可靠性、安全性与卓越运维支柱，本项目主导落地了一套跨可用区（Multi-AZ）双活、纵深安全防御、动静解耦且具备自愈能力的企业级云架构解决方案。

---

## 二、 架构拓扑设计 (Architecture Topology)

```mermaid
graph TD
    Client(["Internet 公网访客"]) -->|HTTP: 80| ALB["Application Load Balancer"]
    
    subgraph VPC ["Amazon VPC (10.0.0.0/16)"]
        subgraph PublicSubnets ["Public Subnets (跨可用区)"]
            ALB -->|Round Robin| EC2_1["EC2 Node 1: 10.0.2.56 (ap-northeast-1c)"]
            ALB -->|Round Robin| EC2_2["EC2 Node 2: 10.0.1.206 (ap-northeast-1a)"]
        end
        
        subgraph PrivateSubnets ["Private Subnets (私有子网组)"]
            EC2_1 -->|MySQL 3306| RDS[("RDS MySQL 8.4: enterprise-db")]
            EC2_2 -->|MySQL 3306| RDS
        end
    end

    EC2_1 -.->|IAM STS 临时凭证| S3["S3 存储桶: enterprise-web-assets"]
    EC2_2 -.->|IAM STS 临时凭证| S3
    
    ALB -.->|UnHealthyHostCount >= 1| CW["CloudWatch Alarms"] -->|Trigger| SNS["SNS 邮件通知"]

```

## 三、 基础设施与配置参数规范 (Infrastructure Matrix)

### 1. 网络与子网拓扑 (Network Topology)
- **VPC**：`enterprise-web-vpc`（CIDR: `10.0.0.0/16`，启用 DNS 主机名与 DNS 解析）
- **公有子网（前端接入层）**：
  - `enterprise-public-sub-1`：`10.0.1.0/24` | 位于可用区 `ap-northeast-1a`（绑定 IGW 路由）
  - `enterprise-public-sub-2`：`10.0.2.0/24` | 位于可用区 `ap-northeast-1c`（绑定 IGW 路由）
- **私有子网（数据持久化层）**：
  - `enterprise-private-sub-1`：`10.0.11.0/24` | 位于可用区 `ap-northeast-1a`（无公网路由）
  - `enterprise-private-sub-2`：`10.0.12.0/24` | 位于可用区 `ap-northeast-1c`（无公网路由）

### 2. 纵深防御网络安全组矩阵 (Security Group Chaining)

| 安全组标识 | 关联对象 | 流量方向 | 协议/端口 | 来源/目标 (Source/Dest) | 设计意图与安全防护逻辑 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **enterprise-alb-sg** | ALB 负载均衡器 | Inbound | TCP 80 | `0.0.0.0/0` | 系统面向互联网的唯一统一合法接入网关。 |
| **enterprise-web-sg** | EC2 Web 集群 | Inbound | TCP 80 | **`enterprise-alb-sg` (SG ID)** | **链式授权**：仅放行来自 ALB 的反向代理流量，直接阻断公网请求。 |
| | | Inbound | TCP 22 | `3.112.23.0/29` (Tokyo EC2 Connect) | 仅允许 AWS 官方管控网段连接，杜绝全网 SSH 暴力破解。 |
| **enterprise-rds-sg** | RDS MySQL 实例 | Inbound | TCP 3306 | **`enterprise-web-sg` (SG ID)** | **数据层白名单**：仅允许挂载了 Web 安全组的计算节点发起 SQL 查询。 |
| | | Inbound | All | 拒绝所有公网访问 (`Publicly Accessible: NO`) | 物理隔离核心数据资产。 |

---

## 四、 核心架构技术实现 (Technical Implementation)

### 1. 计算层无状态化与元数据自愈 (Stateless Compute)
- 实例均通过编写预置的 Cloud-init **User Data** 脚本进行引导部署，避免手工登入配置。
- 在引导阶段集成 **IMDSv2 (Instance Metadata Service v2)**，动态探测机器私有 IP 与可用区信息生成专属响应页，为后续构建 Auto Scaling Group（ASG）奠定无状态计算标准。

### 2. 托管数据层与动静分离 (Data Tier & Storage)
- **RDS for MySQL 部署（成本优先型设计）**：
  - 实例标识符：`enterprise-db`（MySQL 8.4，实例规格 `db.t3.micro`）
  - 架构模式：**单可用区部署（Single-AZ）**。在当前的阶段性验证中，此设计优先满足成本安全；核心生产建议开启 Multi-AZ 同步备机。
  - 专用 DB 子网组：`enterprise-db-subnet-group`（划分于 2 个独立私有子网，具备后续无缝升级 Multi-AZ 的底层网络条件）。
  - 安全隔离：开启 KMS 静态加密，彻底关闭公网访问权限（`Publicly Accessible: NO`）。
  - **内网联调与持久化测试：**
![RDS 跨子网内网查询验证](02-rds-mysql-connection.png)
- **S3 对象存储与免密钥认证 (Zero-Hardcoded Credentials)**：
  - 创建私有存储桶 `enterprise-web-assets-202610`，开启 Block Public Access。
  - 创建 IAM Role `enterprise-ec2-s3-role` 绑定至 EC2 实例配置文件（Instance Profile）。
  - 应用依托底层 AWS STS 临时安全凭证访问 S3，杜绝了 Access Key/Secret Key 硬编码泄露风险。
  - **STS 临时凭证与 S3 免密交互：**
![IAM Role STS 免密鉴权验证](03-iam-sts-s3-upload.png)

---

## 五、 混沌工程验证与容灾测试报告 (Chaos Engineering & Results)

### 实验 1：公网攻击面收敛验证 (Perimeter Security Verification)
- **测试方法**：
  在外部网络环境下，绕过负载均衡器，直接向 Web 节点公网 IP 发送 HTTP 请求，以及使用 PowerShell 向 RDS Endpoint 探测端口：
  `Test-NetConnection enterprise-db.cnuy08kw8rdo.ap-northeast-1.rds.amazonaws.com -Port 3306`
- **实验结果**：
  - 直连 EC2 提示 `Connection Timed Out`（安全组直接将公网非法握手包丢弃）。
  - RDS 连接探测返回 `TcpTestSucceeded : False`，私有隔离验证通过。
  - ALB 统一域名仍可秒级分发至 Node 2 与 Node 1。
  - **测试证据：**
![安全组阻断与合法代理对比](01-security-group-defense.png)

### 实验 2：Web 节点故障注入与流量故障转移测试 (Fault-Injection & Traffic Failover)
- **测试方法**：
  登录位于 `ap-northeast-1c` 的 `enterprise-web-1`，主动停止 Nginx 进程以模拟应用层崩溃：
  ```bash
  sudo systemctl stop nginx
- **观测指标与自愈结果**：
  1. **ALB 目标组判定**：连续健康检查失败，在 30 秒内将 Node 1 状态置为 `Unhealthy (Health checks failed)` 并停止分配新流量。
  2. **终端访问表现**：客户端持续请求网站，流量 100% 自动平滑路由至健康的 Node 2（`ap-northeast-1a`），业务达成 **零中断（Zero Downtime）**。
  3. **CloudWatch 告警触发**：指标 `UnHealthyHostCount` 触碰阈值（蓝线跃升至 1），告警状态由 `OK` 翻转为 `In alarm`。
  4. **SNS 自动化通知**：联动 Amazon SNS 在 60 秒内向架构师/运维邮箱自动投递故障警报邮件。
  5. **故障感知与监控指标触发：**
![目标组健康检查失效剔除节点](04-target-group-unhealthy.png)
![CloudWatch 监控指标越限报警](05-cloudwatch-alarm-triggered.png)

---
## 六、成本优化与资源治理（FinOps）

### 6.1 成本控制

- **AWS Budgets 告警**：配置预算告警，计划使用 $1 和 $3 作为成本监控阈值，并结合预测性告警及时发现潜在超支风险。告警用于提醒，不代表能够自动阻止所有费用产生。
- **网络架构降本**：实验环境将 EC2 部署在公有子网，避免使用 NAT Gateway 产生的额外费用。NAT Gateway 的实际成本取决于区域、运行时长和流量，具体节省金额以实际计费为准。
- **静态资源分离**：将静态媒体资源存储在 S3，减少 Web 实例承担的静态资源存储需求。
- **实例与存储选型**：实验环境优先考虑成本较低的实例和存储配置，例如 `t2.micro`、`db.t3.micro` 和 20 GiB 数据库存储。具体费用及免费额度适用情况需以 AWS 当前政策和实际账户资格为准。

### 6.2 资源清理

实验结束后，按照资源依赖关系逐项检查并清理不再使用的 AWS 资源，避免因遗漏资源而产生持续费用。清理范围包括 EC2、ALB、RDS、EBS、S3 中不再需要的数据及其他已创建的相关资源。删除前应确认是否需要保留数据、日志和配置记录。

## 七、设计权衡与局限性

### 7.1 数据库可用性与成本权衡

本项目使用 RDS Single-AZ 部署 MySQL，主要考虑到个人实验环境的成本限制。Web 层通过 ALB 和跨可用区部署的 EC2 实例提高服务可用性，但数据库层仍存在单点故障风险，因此当前架构尚未实现端到端的跨可用区高可用。

后续可以根据业务的可用性要求和预算，将 RDS 调整为 Multi-AZ，并进一步验证数据库故障切换行为。

### 7.2 EC2 网络部署与安全设计

将 EC2 部署在公有子网可以减少 NAT Gateway 带来的额外费用，但也需要关注公网访问风险。安全组应遵循最小权限原则：仅开放必要的业务端口，限制管理访问的来源，并通过安全组规则限制数据库的访问来源，使其仅接受应用层所需的流量。

对于生产环境，可以进一步评估将 EC2 迁移至私有子网，并采用受控的管理通道，以减少实例直接暴露于公网的风险。

### 7.3 故障检测与流量切换

当 ALB 的健康检查判定某个目标实例不健康时，可以停止向该目标转发新请求，并将流量转发至其他健康目标。这属于故障目标摘除与流量故障转移，并不意味着 EC2 实例本身会自动恢复或被替换。

若需要自动替换故障实例，可以进一步结合 Auto Scaling Group（ASG）实现，并补充相应的故障恢复测试。

### 7.4 测试范围与后续改进

项目的测试结论以实际执行并留存的测试记录为准。对于已经验证的功能，可以记录测试步骤、观察结果和相关截图；对于尚未验证的场景，应明确列为后续改进计划，避免将设计目标描述为已经实现的功能。

后续改进方向包括：

- 将 RDS 升级为 Multi-AZ，验证数据库故障切换。
- 评估 EC2 私有子网部署及受控管理访问方案。
- 根据实际需求配置 ASG，实现故障实例的自动替换。
- 持续检查预算告警、资源清理流程及实际费用。
