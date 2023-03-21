---
title: "About"
date: 2023-03-22T12:12:51+08:00
draft: false
toc: true
images:
tags: 
  - untagged
---

<center>
     <h1>陈洪滨</h1>
     <div>
         <span>
             <img src="assets/phone-solid.svg" width="18px">
             13265156840
         </span>
         ·
         <span>
             <img src="assets/envelope-solid.svg" width="18px">
             13265156840@163.com
         </span>
         ·
         <span>
             <img src="assets/rss-solid.svg" width="18px">
             <a href="https://whocanfly.gitee.io">My Blog</a>
         </span>
     </div>
 </center>

## <img src="assets/info-circle-solid.svg" width="30px"> 个人信息
<img class="photo" style="float:right" src="assets/photo.jpg" width="100px" >
  
- 男，**1996** 年出生  
- 求职意向：后端开发工程师                 
- 工作经验：**3** 年    
- 当前状态：在职

## <img src="assets\graduation-cap-solid.svg" width="30px"> 教育经历

- 学士，广州大学，网络工程专业，**2016.9 ~ 2020.7**
- 奖学金： **2017.09**    广州大学三等奖学金
- CET 4 :   **498**

## <img src="assets/briefcase-solid.svg" width="30px"> 工作经历
+ **2019.10 ~ 2020.02&emsp;&emsp;映客直播&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;电商部&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;服务端开发实习生** 
  
+ **2020.07 ~ 2022.05&emsp;&emsp;中国联通软件研究院&emsp;&emsp;平台架构部云计算组&emsp;&emsp;后端开发工程师**

+ **2022.05 ~ 至今 &emsp;&emsp;&emsp; 中国联通软件研究院&emsp;&emsp;客服研发部智能中心组&emsp;后端开发工程师**

## <img src="assets/project-diagram-solid.svg" width="30px"> 项目经历

**中国联通软件研究院 - 智慧巡场解决方案**&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;**2022.08 ~ 2023.03**

**项目描述：** 以视觉算法驱动、视频汇聚平台为核心的智慧监控巡场解决方案，主要用于智能监控管理中国联通话务中心在岗客服人员的不良行为，降低客服人员的懈怠几率，借助视觉算法精准检测出违规、危险行为，并通知到相关人员、解放视频监控管理人员的大部分精力

**项目标签：** **Java**、**SpringBoot**、**ZLMediakit**、**GB28181**

**主要工作:**

- 主导整体架构设计以及后端开发工作
- 基于优秀开源项目 **Zlmediakit** 作为后端的流媒体服务，提供视频流转码按需转协议、按需推拉流、先播后推、断连续推等功能，以及丰富的 **restful api** 以及 **web hook** 事件
- 基于优秀开源项目 **WVP-PRO** 作为信令服务器、使用 **GB28181** 协议接入管理 **IPC、NVR**
- 基于事件模型集成定制化的 **AI** 视觉算法能力
- 实现从源码编译到 **Kubernetes** 的自动化部署脚本，提升开发测试效率
- 其他功能：
  - 事件处理工作台
  - 物证留存管理
  - 多路视频监控画面点播
  - 白名单管理
  - 监控检测配置
  

----- 

**中国联通软件研究院 - ETCD 集群管理平台**&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp; **2022.03 ~ 2022.04** 

**项目描述**：旨在纳管联通集团所有的 **ETCD** 集群，实现  **ETCD** 集群的自动化运维能力

**项目标签：** **Golang**、**Kubernetes**、**ETCD** 

**主要工作:**

- 主导 **ETCD** 集群管理平台的需求分析和架构设计
- 调研业界 **ETCD** 集群管理的开源方案，并吸取优秀的设计融入到项目中
- 基于 **Kubernetes** 的 **Operator** 二次开发实现了 **ETCD** 集群的创建、销毁功能

---

**中国联通软件研究院 - 云虚拟机平台**&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp; **2020.12 ~ 2021.12**

**项目描述**: 主要用于支撑无法容器化的业务团队, 具备虚拟机申请、资源扩缩容、在线添加磁盘、迁移、快照、克隆等能力，帮助业务团队在分钟级内创建可用的云虚拟机

**项目标签：** **Golang**、**Kubernetes**、**Mesos**、**ETCD**、**Libvirt** 、**QUME-KVM** 

**主要工作:**

- 参与整个项目的架构演进，通过 **Action** 机制和 **json-patch** 减少接口的开发工作
- 基于 **Mesos** 的二次开发实现了虚拟机的任务调度模块
- 充分借鉴了 **Mesos** 开发原则和 **Kubernetes** 的设计思想，即声明式 **API** 和 基于 **Informer** 机制实现控制器模式
- 基于 **ETCD** 实现了主备选举能力， 使用一主两从的方式部署，从而实现高可用
- 部署、维护 **8** 个区域的 **ETCD** 集群

---

## <img src="assets/tools-solid.svg" width="30px"> 技能清单

- 熟悉 **Docker**、**Kubernetes**、**Mesos** 等容器相关领域，拥有相关的工作经验

- 熟悉 **Java、Golang** 等编程语言，有比较扎实的计算机基础知识以及编程技能

- 良好的编码规范，熟悉 **Git、Vim** 等工具的使用，有较好的团队开发意识

- **CNCF**  认证 **Kubernetes** 管理员 **CKA** 证书

- 中国联通软件研究院新员工优秀导师


