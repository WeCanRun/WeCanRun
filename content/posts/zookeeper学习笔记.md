---
title: "Zookeeper学习笔记"
date: 2020-12-20T17:54:51+08:00
draft: false
toc: true
images:
tags: 
  - untagged
---


## zookeeper 的 ACL 权限控制

​       ACL 权限控制，使用 scheme : id : permission 来标识，主要涵盖三个方面：

* 权限模式：授权的策略

* 授权对象：授权对象

* 权限：授予的权限

  其特性如下：

* zookeeper 的权限控制是基于每个znode节点的，需要对每个结点设置权限

* 每个znode支持设置多种权限方案和多个权限

* 子节点不会继承父节点的权限，客户端无权访问某节点，但可能可以访问它的子节点

e.g.

```shell
setAcl /test2 ip:192.168.60.130:crwda 
```



### 权限模式

​          采用何种方式授权

| 方案   | 描述                                              |
| ------ | ------------------------------------------------- |
| world  | 只有一个用户：anyone，代表登录zookeeper所以人（） |
| ip     | 对客户端使用IP地址认证                            |
| auth   | 使用已添加的用户认证                              |
| digest | 使用 ”用户名：密码“ 方式认证                      |



### 授权对象

​	给谁授予权限

​	授权对象ID是指，权限赋予的实体，例如：IP地址或用户



### 授予的权限

​	授予什么权限

​	可以授予的权限包括 create、delete、read、write、admin 也就是增、删、改、查、管理权限，这5种权限简写为cdrwa，注意这5种权限中，delete是指对子节点的删除权限、其他四种指对自身的操作权限

| 权限   | ACL 简写 | 描述                             |
| ------ | -------- | -------------------------------- |
| create | c        | 可以创建子节点                   |
| delete | d        | 可以删除子节点                   |
| read   | r        | 可以读取节点数据及显示子节点列表 |
| write  | w        | 可以设置节点数据                 |
| admin  | a        | 可以设置节点访问控制列表权限     |



### 授权的相关命令

| 命令   | 使用方式     | 描述 |
| ------ | ------------ | ---- |
| getAcl | getAcl<path> | 读取ACL.权限 |
| setAcl | setAcl<path><acl> |设置ACL权限|
| addauth | addauth<scheme><auth> | 添加认证用户 |



### 案例

* world 授权模式

  命令

  ```shel
  setAcl <apth> world:anyone:<acl>
  ```



* IP 授权模式

  命令

  ```shell
  setAcl <path> ip:<ip>:<acl>
  ```



* auth 授权模式

  命令

  ```shell
  addauth digest <user>:<password> # 添加认证用户
  setAcl <path> auth:<user>:<acl>
  ```

  e.g.

  ```shell
  [zk: 172.17.0.4:2181(CONNECTED) 1] create /node "node"
  Created /node
  [zk: 172.17.0.4:2181(CONNECTED) 2] getAcl /node
  'world,'anyone
  : cdrwa
  [zk: 172.17.0.4:2181(CONNECTED) 3] setAcl /node auth:wuyi:adra
  Acl is not valid : /node
  [zk: 172.17.0.4:2181(CONNECTED) 4] addauth digest wuyi:123456
  [zk: 172.17.0.4:2181(CONNECTED) 5] setAcl /node auth:wuyi:adra
  [zk: 172.17.0.4:2181(CONNECTED) 6] getAcl /node
  'digest,'wuyi:dXM1WQjCmiOQtOD8PAQtDp5fCpY=
  : dra
  [zk: 172.17.0.4:2181(CONNECTED) 7] 
  ```



* digest 授权模式：

  命令

  ```shell
  setAcl <path> digest:<user>:<password>:<acl>
  ```

  这里的密码是经过 SHA1及 base64 处理的密文，在 shell 中可以通过一下命令计算：

  ```shell
  echo -n <user>:<password> | openssl dgst -binary -sha1 | openssl base64
  ```

  e.g.

  ```shell
  [root@centos7 ~]# echo -n wuyi:123456 | openssl dgst -binary -sha1 | openssl base64
  dXM1WQjCmiOQtOD8PAQtDp5fCpY=
  ```

  ```shell
  [zk: 172.17.0.4:2181(CONNECTED) 1] create /node "node"
  Created /node
  [zk: 172.17.0.4:2181(CONNECTED) 2] setAcl /node digest:wuyi:dXM1WQjCmiOQtOD8PAQtDp5fCpY=:cdrwa
  [zk: 172.17.0.4:2181(CONNECTED) 3] getAcl /node
  Authentication is not valid : /node
  [zk: 172.17.0.4:2181(CONNECTED) 4] addauth digest wuyi:123456
  [zk: 172.17.0.4:2181(CONNECTED) 5] getAcl /node
  'digest,'wuyi:dXM1WQjCmiOQtOD8PAQtDp5fCpY=
  : cdrwa
  [zk: 172.17.0.4:2181(CONNECTED) 6] 
  ```

  

  ### ACL 超级管理员

  ​	zookeeper  的权限管理模式有一种叫做 super，该模式提供一个超管可以方便的访问任何权限的节点，假设这个超管是 super:admin，需要先为超管生成密码的密文

  ```shell
  echo -n super:admin | openssl dgst -binary -sha1 | openssl base64
  ```

  ​	在 zookeeper 目录 /bin/zkServer.sh 中添加如下代码

  ```shell
  "Dzookeeper.DigestAuthenticationProvider.superDigest=super:xQJmxLMiHGwaqBvst5y6rkB6HQs=" \
  ```

   添加之后为

  ![image-20200831101424059](https://s2.loli.net/2022/04/10/E64NWqnwzLbTrRP.png)

  重启 zookeeper 之后

  ```shell
  addauth digest super:admin # 添加认证用用户
  ```

  



