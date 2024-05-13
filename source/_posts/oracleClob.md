---
title: oracle转化字段为clob
date: 2017-08-23 20:58:05
tags: [oracle]
categories: Oracle
comment: true
---

因为Oracle是没办法直接进行字段转化的,所以这里我们提供一个取巧的办法:

``` bash
    --增加大字段项  
    alter table 需要修改的表 add introduce clob;  
    --将需要改成大字段的项内容copy到大字段中  
    update 需要修改的表 set introduce=需要修改的字段;
    --删除原有字段  
    alter table 需要修改的表 drop column 需要修改的字段;  
    --将大字段名改成原字段名
    alter table t_sim_activity rename column introduce to 需要修改的字段;
```