+++
draft = true
date = 2024-03-17T01:22:23+08:00
title = "个人简历"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
disableComments = true
+++


## 王颖

| 性别 | 学历 | 出生年份 | 联系电话     | 邮箱             |
|:---- |:---- |:-------- |:------------ |:---------------- |
| 男   | 本科 | 1993年   | 187-7296-4832  | rifewang@gmail.com |

主导过的互联网 SaaS 项目处理**日均千万级** PV 流量、管理**数十亿**图片资源。
负责研发的脑科学云平台及医疗器械项目服务于几十家头部医院、科研机构和诊所。

丰富的技术广度，拥有 **系统架构设计师**、**Kubernetes**、**ElasticSearch** 等官方技术认证，涵盖 **后端**、**容器与云原生**、**搜索**、**大数据** 等多项技术领域。

职场表现卓越，每一段生涯都能做出优秀成绩，曾获：**年度优秀个人**、**年度创新团队** 等荣誉。
坚持运动、学习、写作 8 年，公众号：[系统架构师Go](https://raw.githubusercontent.com/RifeWang/images/master/qrcode.jpg)，技术社区 Segmentfault 2022、2023 年度 Maintainer。
在掘金创作有：[Kubernetes 云原生](https://juejin.cn/column/7314642642869403682)、[AI 人工智能](https://juejin.cn/column/7425885062921928738)、[ELK 搜索与大数据](https://juejin.cn/column/7314860085930180623) 等多个专栏。

## 工作经历

<h3 style="background-color: #87CEFA; padding: 10px;">
  优脑银河（浙江）科技有限公司
  <span style="float: right;">2021.07 ~ 2023.12</span>
</h3>

#### 前沿脑科学云平台（ https://app.neuralgalaxy.cn/ ）

项目概述：针对自闭症、抑郁症、失语、运动障碍等多种脑疾病的科研与诊疗**云平台**，为科研和医疗机构提供患者管理、影像数据管理、个体精准脑图谱绘制、任务编排、离线在线计算、智能诊疗等服务。

项目成绩：定制化的多模态阅片产品成功交付某部战区总医院；科研云平台支撑了多次大规模临床试验及日常科研；疗法云软件投入商业化运作并持续迭代。

个人职责：
- Team Leader，实施 Scrum 敏捷开发，组织 code review，协调团队成员工作，安排产品的上线发版。
- 架构设计和演化，包含技术选型、制定架构演进方案、组织技术评审、追踪并确保技术落地。
- 日常编码实现业务需求，包括 RESTful API、MySQL 建模、Redis 缓存、MQ 消息队列、S3/Minio 对象存储、NAS/Ceph 文件存储、Serverless 集成、Puppeteer 无头浏览器截图，以及医学影像处理系统和算法集成。
- 系统优化，深入整体架构、业务流、数据流、各组件等多角度优化系统，保障系统的高性能、可扩展、高质量。
- 构建 Prometheus、Grafana 可观测性系统，采集日志、追踪、指标，搭建数据报表与可视化数据分析大屏。
- 实施 DevOps 和云原生，负责 Kubernetes 基础设施、Argo-workflows 任务编排引擎、 Terraform IaC、以及等其它云原生工具。

<h3 style="background-color: #87CEFA; padding: 10px;">
  杭州又拍云科技有限公司
  <span style="float: right;">2018.04 ~ 2021.04</span>
</h3>

#### 又拍图片管家（ https://x.yupoo.com ）
项目概述：互联网 SaaS Web 应用，为中小商家提供图片视频的管理、CDN 外链加速、个性化主页展示、站内搜索、统计分析等功能。

项目成绩：在职期间注册**用户数增长 6 倍**，管理图片资源数量**增长 5 倍**，获得年度创新团队。

个人职责：
- 负责 Web 主站的架构迭代和后端研发工作，处理每日千万级 PV 访问、管理十亿级图片资源。
- 确保系统应对高并发挑战，保障系统的高可用和高性能。
- 设计并实现 RESTful API 接口，使用 Cache Aside Pattern 构建分布式 Redis 集群缓存。
- MySQL 数据建模及优化，分库分表拆分 2 亿行数据大单表，调整索引优化数据库性能。
- 推动系统架构向微服务演进，利用消息队列 NSQ / Kafka 削峰填谷、解耦并异构服务，保障系统稳定性。
- 引入 InfluxDB 分流存储时序数据，构建 ClickHouse 数据仓库，搭建 OLAP 日志分析系统。
- 构建 ElasticSearch 全文搜索服务、Milvus 向量搜索服务，搭建 Grafana 可视化数据报表系统。

#### AI 以图搜图系统
项目概述：提供图像内容进行相似性搜索。

个人职责：从零探索，并完成两次迭代，构建了以 Embedding 和 Milvus 向量数据库为核心的搜图系统。此项目被广泛参考，并成为**业界标杆**：[《又拍图片管家亿级图像之搜图系统的两代演进及底层原理》](https://segmentfault.com/a/1190000022842774)。

#### 大数据处理系统
项目概述：对 Web 主站访问和用户图片 CDN 流量日志（**千万/日**）进行采集处理、统计分析、数据报表。

个人职责：对 Web 主站访问数据实施 kafka->logstash->elasticsarch 的 ETL 过程。对用户图片 CDN 流量日志，将 Hadoop 替换为了以 ClickHouse 为核心的 OLAP 系统，完成了实时流量计费，并提供数据报表。

<h3 style="background-color: #87CEFA; padding: 10px;">
  财游（上海）信息技术有限公司
  <span style="float: right;">2017.04 ~ 2018.04</span>
</h3>

项目：财宝理财（互联网金融 P2P 项目）。

个人职责：从零将产品打造上线，负责全栈开发、系统架构、基础设施等工作。获得年度优秀个人。

<h3 style="background-color: #87CEFA; padding: 10px;">
  武汉东浦信息技术有限公司
  <span style="float: right;">2016.06 ~ 2017.04</span>
</h3>

项目：汽车保养预约服务。

个人职责：从零构建项目，负责全栈开发，成为公司模板项目。

## 专业技能

- 8 年开发 5 年架构，较强的架构和编码能力，对高并发、高可用、高性能、分布式、微服务有深入理解和实践。
- 具备技术团队管理经验，拥有 Scrum 敏捷开发、DevOps、CICD、云原生等实践经验。
- 擅长 Golang、Node.js，熟悉 Python，能够快速上手其它语言，并灵活运用多种语言解决挑战性难题。
- 熟悉 MySQL、Redis、MQ、Elasticsearch 等常用组件原理，具备全文搜索和日志聚合分析经验。
- 熟悉 Kafka、ClickHouse、ELK、Prometheus、Grafana，拥有数据仓库、BI 和大数据领域项目经验。
- 熟悉 Linux，理解 Docker、Kubernetes、Argo 架构及原理，Kubernetes 文档贡献者，熟悉云原生周边工具。
- 熟悉 CICD，包括 ArgoCD、GitHub Actions、Terraform 等工具。
- 熟悉医学影像处理、计算机视觉、Milvus 向量搜索。理解 LLM 等 AI 基本原理，日常深度使用 AI 以提升效率。

## 致谢

感谢您阅读我的简历，期望能携手成就一番事业！