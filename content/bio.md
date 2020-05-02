---
title: "关于我"
date: 2020-01-1T10:45:47+08:00
draft: false
featured_image: https://oss.likeli.top/uPic/20200407174219
---

# 联系方式

- 李科笠/男/1992
- Email：<a href="mailto:li.keli0529@gmail.com">li.keli0529@gmail.com</a>

---

# 个人信息

 - 学历：本科
 - 工作年限：6年
 - Github：[https://github.com/li-keli](https://github.com/li-keli)
 - Blog：[https://blog.likeli.top](https://blog.likeli.top)
 - 期望职位：资深Go开发工程师
 - 期望城市：武汉/北京

---

# 工作经历

## 北京金色世纪商旅网络科技股份有限公司 （ 2016年10月 ~ 至今 ）

**职位：** Go高级开发工程师

## 项目经验

### IM推送网关

**项目描述：** 自建IM推送系统，负责企业内部服务对外部的终端（IOS、Android、Web、微信小程序、微信公众号）进行统一的推送服务。项目期间我负责整个推送系统的架构设计和主要编码工作。

* 使用WebSocket协议实现实时推送，支持客户端心跳。
* 使用分布式开发模式来编写无状态的WebSocket连接端，提高连接数，并部署APIGateway（Kong）进行反向代理和统一鉴权。
* 基于Redis中间件实现消息过期，基于其订阅发布实现任务的调度。
* 基于RabbitMQ中间件实现流量的削峰，并保证消息到达。
* 使用ELK（Elasticsearch、Logstash、Kibana）集中收集运行日志，并做可视化分析。
* 使用MongoDB的副本集集群模式避免存储的单点问题。
* 使用Docker Swarm部署服务，实现蓝绿发布。

**技术栈：** Go、Gin、WebSocket、Redis/RabbitMQ、Mongodb、ELK

---

### 厅销售系统

**项目描述：** 此系统是一套APP、两个后台系统组成的销售管理系统。实现全国的内部员工销售、审批、上架、仓储、物流等的线上管理。项目期间负责数据层、处理面向APP的后端API开发，并给内部提供RPC服务，并负责搭建ELK三件套处理日志，通过Kibana做日志可视化分析，实时追踪系统运行情况。

**技术栈：** Go、Gin、ELK、Vue、Docker

---

### 支付网关

**项目描述：** 企业内统一的支付网关，对接数十种支付机构，对内部服务提供统一的支付方式、收款、退款API。提供简化的支付接入能力。项目期间我负责接入微信、银联、哆啦宝等支付机构，对内输出支付方式和收款API，并进行长期维护。在接手项目期间，修复了若干遗留的严重支付逻辑漏洞，并引入了MQ，提升了支付网关系统的健壮性。

**技术栈：** Go、Gin、RabbitMQ、Redis、MySQL

---

### 服务容器化改造

**项目描述：** 改造之前公司内部的服务是高聚合的，会员数据、订单数据、销售系统、机票、酒店等业务服务都是聚合在几个大的项目中，每次代码上线、BUG排查和修复都非常困难。后推行微服务和DevOps，将聚合的大项目进行拆分，并进行容器化部署、上CI/CD。我负责此次的部分项目拆分、并迁移到Dubbo生态，另外探索容器化部署方案（Docker）、服务编排（Docker Swarm）、CI/CD等。容器化改造后，开发和交付效率成倍的提升，释放了开发大量的时间和精力，代码质量也有不错的提升。

* 负责编写内部的实践文档.
* 项目使用DevOps实践后，各个开发组成员效率成倍提高，避免了人肉部署的繁琐和失误，不用再关心服务器环境差异。
* 利用Docker Swarm发布，实现了蓝绿发布。
* 服务的API文档自动发布到YAPI文档中心，实现了文档的统一和权限管理。
* 带领项目组逐步尝试进行微服务化

**技术栈：** Docker、Docker-Compose、Docker Swarm、Gitlab、CI/CD、YAPI

---

## 北京CBD商务中心区通信科技有限公司 （ 2014年10月 ~ 2016年10月 ）

**职位：** 软件开发工程师

### 企业数据爬虫

采集北京市工商信息网的企业数据进行可视化分析。我负责爬虫的设计开发工作。

* 使用requests库做网络请求
* 使用pyquery做数据摘取
* 使用RabbitMQ的Work模式实现分布式
* 使用MongoDB存储
* 使用Tor网络突破IP限制
* 使用SVM处理图形验证码

**技术栈：** Python、requests、pyquery、xpath、MongoDB、RabbitMQ

---

# 技能清单

以下均为我熟练使用的技能

- 主力语言：Go/NetCore/JAVA
- 脚本：Python
- Web框架：Gin/SpringBoot/AspnetCore
- 中间件：Redis/RabbitMQ/Nginx
- 前端框架：Vue
- 数据库相关：Mongodb/MySQL/MyBatis/Gorm
- 容器化：Docker/Docker Swarm/Docker-Compose
- 其他：Git


# 教育经历

* （2017-2020）本科 北京交通大学-计算机科学与技术
* （2011-2014）专科 湖北国土资源职业技术学院-软件技术

# 致谢
感谢您花时间阅读我的简历，期待能有机会和您共事。