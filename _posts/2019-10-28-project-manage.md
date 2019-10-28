---
layout:     post
title:      "开发流程管理"
subtitle:   "开发流程管理"
date:       2019-10-28
author:     "zark"
header-img: "img/in-post/2016.03/07/post-markdown-introduce.jpg"
tags:
    - 项目开发流程
---

- [技术预研](#技术预研)
- [需求评审PRD](#需求评审)  
- [Kick Off](#KickOff)  
- [设计评审](#设计评审)  
- [开发计划](#开发计划)  
- [TC评审](#TC评审)  
- [提测](#提测)  
- [发布评审](#发布评审)  
- [预发验证、发布](#预发验证、发布)

### 技术预研
线下跟PD沟通，项目可能用到新的技术、或者常规的业务技术。可以提前了解技术可不可行、看看目前市场上竞品是怎么实现的

### 需求评审
了解需求的业务价值、业务方向
需求评审两次不通过，警示第三次必须通过，不然项目暂时先不做。 因需求评审不过，系统发布时间因相应推迟。要是发布时间不能推迟，需给PD压力，开发加班是因为他的工作失职

### KickOff
- 确定资源  
    协调测试、前端、UI、交互资源，确定人是否能参与到项目中

- 确定项目的里程碑  
    什么时候联调、什么时候测试、什么时候上线

- 通报关键时间点  
    邮件通知所有项目成员，知晓项目的关键时间点

### 设计评审
- 架构图  
- 表结构  
- 核心流程时序图  
- 系统难点设计  


### 开发计划
#### 制定开发计划需要注意的点
- 模块里程碑  
    项目里程碑时间需要定义好，对项目有个大体事件的把控

- 预留1周左右buffer  
    防止开发中突发情况，技术难点之类的

- 自测  
    预留自测时间，提高后面的联调效率及减少bug出现

- 联调  
    前后端要是涉及一对多联调，时间要岔开，避免出现同一天多个后端开发找一个前端开发联调；  
    要是涉及到和外部系统联调，需要提前跟外部系统沟通，确定联调时间

- 周末节假日  
    评估时间避免跟周末节假日时间冲突  
    
#### 开发过程中需要注意的点
- 注意点  
    自己开发的模块cover不住（技术难题攻克不了、手头临时别的事情处理），需及时上报，提前一起想办法解决；  
    遇到线上和线下配置不一样的要用小本本记下来，作为发布脚本  

#### 项目搭建  
- 项目结构图 
```text
├── iproject[项目根目录]
|
│   ├── iproject-common[公共逻辑]
|   |
|   |   ├── constants[常量包]
|   |   |
|   |   ├── util[工具包]
|   |
│   ├── iproject-service[业务逻辑]
|   |
|   |   ├── common[公共业务逻辑包]
|   |
│   ├── iproject-dao[DAO\Mapper]
|   |
|   |   ├── entity[实体包]
|   |   |
|   |   ├── mapper[Mapper包]
|   |
│   ├── iproject-api[API接口]
|   |
│   ├── iproject-starter[启动模块]
|   |
│   ├── iproject-web[Web模块]
|   |
│   ├── README.md[项目介绍] 
```  

- 依赖关系
- iproject-starter(项目启动模块，配置一些启动信息及Bean注入&Configuration)
    - iproject-web(Web模块，Controller所在位置)
        - iproject-service(Service模块，具体业务逻辑)
            - iproject-dao(Dao模块，Mapper文件生成位置)
            - iproject-common(Common模块，公共代码位置，不牵扯业务逻辑)
            - iproject-api(二方库，纯接口和Pojo)
 
#### 前后端联调API接口定义
#### 版本变更说明
 
| 日期	    | 版本 	 | 说明	|   作者	     |备注|
| ----------- | ------ | ----------------| ------ | ---|
|2019年07月02日|v0.1.0|初稿| XX |初稿|

#### API名称：获取当前用户可见所有租户信息&报表组信息

```text
GET ${base-domain}/studio_api/tenant
```
#### 描述

```text
获取当前用户可见所有租户信息&每个租户下对应的报表简略信息
```

#### 请求参数

| 请求参数名 | 类型 | 必填 | 传参位置 |说明
| ----------- | ------ | ----------------| ------ | ---
|buc_cookie|String|Y|cookie|用户buc登录cookie
|type|String|Y|url|租户的类型10代表studio 

#### 请求参数示例

```text
GET ${base-domain}/studio_api/tenant
``` 

#### 返回参数

| 返回参数名 | 类型  |说明
| ----------- | ------ | ------------
|buc_cookie|String|用户buc登录cookie
|type|String|租户的类型10代表studio 

#### 返回结果示例

成功返回值

```json
{
    "code":"200",
    "msg":"",
    "data":[{
        "id":1,
        "name":"foo租户",
        "expiredTime":15012313111,
        "subjectGroups":[{
            "id": 1,
            "name": "subjectGroup1",
            "tenantId": 1
        }]
    }]
}
```

失败返回值 

```json
{
    "code":"500",
    "msg":"获取租户失败!",
    "data":""
}
```

**工时评估**
<table>
    <tr>
        <td>模块</td>
        <td>功能点</td>
        <td>工作量</td>
        <td>负责人</td>
        <td>备注</td>
    </tr>
    <tr>
        <td rowspan="2">xxx模块1</td>
        <td>功能点描述1</td>
        <td>1</td>
        <td>张三</td>
        <td>这是个备注</td>
    </tr>
    <tr>
        <td>功能点描述2</td>
        <td>1</td>
        <td>李四</td>
        <td>这是个备注</td>
    </tr>
    <tr>
        <td>xxx模块2</td>
        <td>功能点描述3</td>
        <td>2</td>
        <td>李四</td>
        <td>这是个备注</td>
    </tr>
    <tr>
        <td>前后端联调</td>
        <td>前后端联调</td>
        <td>2</td>
        <td>张三、李四</td>
        <td></td>
    </tr>
        <tr>
        <td>系统Buffer</td>
        <td>系统Buffer</td>
        <td>5</td>
        <td></td>
        <td>预留时间</td>
    </tr>
    <tr>
        <td>合计：</td>
        <td></td>
        <td>11</td>
        <td></td>
        <td></td>
    </tr>
</table>

**甘特图**  
- 第一周
<table>
    <tr>
        <td></td>
        <td>周一</td>
        <td>周二</td>
        <td>周三</td>
        <td>周四</td>
        <td>周五</td>
    </tr>
    <tr>
        <td>关键事件</td>
        <td></td>
        <td></td>
        <td></td>
        <td>kickoff</td>
        <td></td>
    </tr>
    <tr>
        <td>张三</td>
        <td>完成任务1</td>
        <td>完成任务2</td>
        <td>完成任务3</td>
        <td>完成任务4</td>
        <td>完成任务5</td>
    </tr>
    <tr>
        <td>李四</td>
        <td colspan="2" align="center">完成任务1</td>
        <td>完成任务2</td>
        <td>完成任务3</td>
        <td>完成任务4</td>
    </tr>
</table>

### TC评审
- 确定测试case是否覆盖全  
    
- 测试的逻辑是否和开发一致  


### 提测
- 冒烟  
    核心流程走通，所有Bug当天解决。冒烟结果需要邮件通知项目成员

- 测试  
    按优先级解决当天6点前提出的所有bug。确认是否有性能测试需求？并发测试需求？

- 回归测试  
    期间不可以提交代码，要是必须修改代码，需通知测试

- bug工单格式
> [缺陷描述]：  
> [重现步骤]：  
> [期望结果]：  
> [实际结果]：  
> [原因定位]：  
> [修复建议]：   

- bug 状态
> Open - 未解决  
> Fixed - 已解决  
> Won'tfix - 不予解决  
> Later - 之后解决  
> Worksforme - 无法重现  
> Duplicate - 重复问题  
> Invalid - 无效bug  
> External - 外部原因  
> ByDesign - 设计如此  

### 发布评审

主要是确定发布方案是否可行，确认具体回滚措施，如果保证发布成功

#### 需要注意的点
- 发布顺序  
    需要申请机器？表结构变更？Diamond？项目？初始化线上SQL脚本？

- 回滚方案  
    提前确认发布失败的回滚措施，不影响系统使用

- 怎样验证是否发布成功  
    确认功能是否发布上去。eg：跑一遍逻辑

- 外部系统牵连  
    要是系统以来外部系统，需要提前邮件确认外部系统发布时间

- 老数据兼容  
    发布之前是否需要跑脚本初始化老数据？

### 预发验证、发布
- 用户使用预发环境体验系统，期间要是有体验问题，做好记录，下期排期解决
    
- PD跟PM需要到场支持
