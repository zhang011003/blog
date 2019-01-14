# hs-web简介、数据表设计

## 简介

hs-web是一个用于快速搭建企业后台管理系统的基础项目,集成一揽子便捷功能如:通用增删改查，在线代码生成，权限管理，OAuth2.0 ,动态多数据源分布式事务，动态脚本，动态定时任务，在线数据库维护等等. 基于 spring-boot,mybaits。项目地址 [hs-web-framework](https://github.com/hs-web/hsweb-framework)。

## 代码目录结构

该项目的所有功能功能都是通过SpringBoot的starter的方式动态增加的，这点要牢记。

基本目录结构如下（以authorization为例）：

![hs-web项目图片](../../screenshot/hs-web.png)

其中

* hsweb-system-authorization-starter模块为starter模块，
* hsweb-system-authorization-api模块包括service接口和entity实体类，
* hsweb-system-authorization-local模块包括service实现和dao接口，以及mybatis用到的mapper配置文件，
* hsweb-system-authorization-web模块为controller

看懂一个具体的模块，其它模块都是类似的结构，就非常好理解了。

下面先看一下hsweb-framework涉及到的数据表的设计

## hsweb-framework数据表的设计

hsweb-framework共包括77张表，除去工作流表25张以及quartz表11张外，共41张表。

权限相关表如下：

* s_menu为系统菜单表
* s_user为系统用户表
* s_role为角色表
* s_user_role为用户与角色关联表
* s_permission为权限表
* s_permission_role为权限与角色关联表（但实际该表中无数据）
* s_autz_setting为权限设置表（保存设置过的用户id与权限id）
* s_autz_menu为权限设置菜单表（s_autz_setting和s_menu的关联表）
* s_autz_detail为权限设置详情表（s_autz_setting和s_permission的关联表，也保存action（可操作类型）以及data_accesses（数据权限控制），对应到页面用户管理->用户赋权页面 以及 角色管理->角色赋权页面）

按照常规思路，权限设计会用到5张表：用户、角色、权限以及用户角色、角色权限关联关系表。但在hs-web中，可以给用户赋予权限，也可以给角色赋予权限，再给用户赋予角色。所以惯常的5张表设计需要调整。hs-web中是增加了一张s_autz_setting表，这样在关联关系表中，关联的不是角色id或者用户id，而是setting表的id。理解了这一点后，就很好理解hs-web的权限表设计思路了。

后续表在分析到相应功能后再做补充。



