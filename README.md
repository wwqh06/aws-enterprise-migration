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

```text
                                  【 Internet 公网访客 】
                                             │
                                             ▼ HTTP : 80
                                ┌─────────────────────────┐
                                │ Internet Gateway (IGW)  │
                                └────────────┬────────────┘
                                             │
====================================== VPC (10.0.0.0/16) ======================================
  [ Public Route Table: 0.0.0.0/0 -> IGW ]
  
  ┌─────────────────────────────────【 Public Subnets 】─────────────────────────────────────┐
  │                                                                                          │
  │   Availability Zone A (ap-northeast-1a)         Availability Zone C (ap-northeast-1c)    │
  │   ┌───────────────────────────────────┐         ┌───────────────────────────────────┐    │
  │   │ enterprise-public-sub-1           │         │ enterprise-public-sub-2           │    │
  │   │ CIDR: 10.0.1.0/24                 │         │ CIDR: 10.0.2.0/24                 │    │
  │   │                                   │         │                                   │    │
  │   │   ┌───────────────────────────────┴─────────┴───────────────────────────────┐   │    │
  │   │   │         Application Load Balancer (ALB: enterprise-alb)                 │   │    │
  │   │   │         [Security Group: enterprise-alb-sg (Allow Inbound 80)]          │   │    │
  │   │   └───────────────────────────────┬─────────┬───────────────────────────────┘   │    │
  │   │                                   │         │ (Round Robin / Health Checks)     │    │
  │   │                  ┌────────────────┘         └────────────────┐                  │    │
  │   │                  ▼                                           ▼                  │    │
  │   │   ┌──────────────────────────────┐          ┌──────────────────────────────┐    │    │
  │   │   │ enterprise-web-2 (t2.micro)  │          │ enterprise-web-1 (t2.micro)  │    │    │
  │   │   │ Private IP: 10.0.1.206       │          │ Private IP: 10.0.2.56        │    │    │
  │   │   │ SG: enterprise-web-sg        │          │ SG: enterprise-web-sg        │    │    │
  │   │   │ (HTTP 80 ONLY from ALB-SG)   │          │ (HTTP 80 ONLY from ALB-SG)   │    │    │
  │   │   │ IAM: enterprise-ec2-s3-role  │          │ IAM: enterprise-ec2-s3-role  │    │    │
  │   │   └──────────────┬───────────────┘          └──────────────┬───────────────┘    │    │
  │   └──────────────────┼─────────────────────────────────────────┼────────────────────┘    │
  └──────────────────────┼─────────────────────────────────────────┼─────────────────────────┘
                         │                                         │
                         │ MySQL Port: 3306 (Private VPC Traffic)  │
                         └────────────────┬────────────────────────┘
                                          │
  ┌──────────────────────────────【 Private Subnets 】───────────────────────────────────────┐
  │                                       ▼                                                  │
  │   Availability Zone A (ap-northeast-1a)         Availability Zone C (ap-northeast-1c)    │
  │   ┌───────────────────────────────────┐         ┌───────────────────────────────────┐    │
  │   │ enterprise-private-sub-1          │         │ enterprise-private-sub-2          │    │
  │   │ CIDR: 10.0.11.0/24                │         │ CIDR: 10.0.12.0/24                │    │
  │   │                                   │         │                                   │    │
  │   │   ┌───────────────────────────────┴─────────┴───────────────────────────────┐   │    │
  │   │   │               RDS MySQL 8.4 Instance (enterprise-db)                    │   │    │
  │   │   │               [DB Subnet Group: enterprise-db-subnet-group]             │   │    │
  │   │   │               [SG: enterprise-rds-sg (Allow 3306 ONLY from Web-SG)]     │   │    │
  │   │   │               [Publicly Accessible: NO]                                 │   │    │
  │   │   └─────────────────────────────────────────────────────────────────────────┘   │    │
  │   └─────────────────────────────────────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────────────────────────────────┘
==============================================================================================

  ┌─────────────────────────┐               ┌────────────────────────────────────────────────┐
  │ Amazon S3 Bucket        │ ◄──(IAM STS)──┤ Amazon CloudWatch & SNS                        │
  │ enterprise-web-assets-  │ (Zero-Key)    │ • Metric: UnHealthyHostCount >= 1              │
  │ 202610                  │               │ • Action: Automated Email Alert Triggered      │
  └─────────────────────────┘               └────────────────────────────────────────────────┘
