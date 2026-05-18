---
title: Oracle CASE WHEN 判断 NULL 的一个坑
date: 2026-05-18 21:00:00
tags:
  - Oracle
  - SQL
keywords: Oracle CASE WHEN NULL
categories: 实操
---



# Oracle CASE WHEN 判断 NULL 的一个坑

今天写报表 SQL 的时候遇到一个问题，记录一下。

场景大概是这样的，门店有供货方类型字段 `VC2SUPPLIER1TYPE`，正常情况下：

* `1`：经销商
* `2`：分销商
* `3`：仓库

然后根据不同类型，去不同表里取供货方编码、供货方名称。

但是有一批历史数据，`VC2SUPPLIER1TYPE` 是空的，不过 `VC2SUPPLIER1CODE` 里存的是一个真实的 ID。比如这条数据里：

```text
VC2SUPPLIER1TYPE = NULL
VC2SUPPLIER1CODE = ff8080819ca9d147019ca9d167830598
```

再单独查经销商表：

```sql
SELECT *
FROM MM_DEALER
WHERE ID = 'ff8080819ca9d147019ca9d167830598';
```

是能查到数据的，并且经销商编码是：

```text
111041
```

所以我原本的想法是：

* 如果供货方类型是 `1`，直接走经销商关联
* 如果供货方类型是 `2`，直接走分销商关联
* 如果供货方类型是 `3`，直接走仓库关联
* 如果供货方类型是 `NULL`，就按 ID 依次去经销商、分销商、仓库表里兜底查一下

但是实际查出来的时候，供货方编码还是这个原始 ID：

```text
ff8080819ca9d147019ca9d167830598
```

并没有变成：

```text
111041
```

问题就出在 `CASE WHEN NULL` 这里。



## 问题 SQL

当时大概写的是这种：

```sql
CASE t1.VC2SUPPLIER1TYPE
    WHEN '1' THEN store_supplier_dealer.VC2ID
    WHEN '2' THEN store_supplier_distributor.VC2CODE
    WHEN '3' THEN store_supplier_warehouse.VC2WHSENUM
    WHEN NULL THEN COALESCE(
        (SELECT d2.VC2ID
           FROM MM_DEALER d2
          WHERE d2.ID = t1.VC2SUPPLIER1CODE),
        (SELECT dt2.VC2CODE
           FROM MM_DISTRIBUTOR dt2
          WHERE dt2.ID = t1.VC2SUPPLIER1CODE),
        (SELECT w2.VC2WHSENUM
           FROM MM_WAREHOUSE w2
          WHERE w2.ID = t1.VC2SUPPLIER1CODE),
        t1.VC2SUPPLIER1CODE
    )
    ELSE t1.VC2SUPPLIER1CODE
END AS 供货方编码
```

看起来好像没啥问题：

```sql
WHEN NULL THEN ...
```

肉眼看就是，如果 `t1.VC2SUPPLIER1TYPE` 是空，就走后面的子查询。

但 Oracle 实际不是这么判断的。



## CASE 有两种写法

这里就顺便把 `CASE` 的两种写法记一下。

### 1. 简单 CASE

这种叫简单 CASE：

```sql
CASE 字段
    WHEN 值1 THEN 结果1
    WHEN 值2 THEN 结果2
    ELSE 默认结果
END
```

例如：

```sql
CASE supplier_type
    WHEN '1' THEN '经销商'
    WHEN '2' THEN '分销商'
    WHEN '3' THEN '仓库'
    ELSE '未知'
END
```

这个可以简单理解为：

```text
supplier_type = '1'
supplier_type = '2'
supplier_type = '3'
```

也就是说，`CASE supplier_type` 后面的每个 `WHEN`，本质上都是拿 `supplier_type` 去跟它做等值比较。

这种写法适合状态、类型这种固定值转换。

比如：

```sql
CASE cooperation_status
    WHEN '1' THEN '合作'
    WHEN '0' THEN '不合作'
    ELSE '未知'
END
```

简单、好看，也够用。

但是它有一个很容易忽略的问题：不要在这里写 `WHEN NULL`。



### 2. 搜索式 CASE

另一种叫搜索式 CASE：

```sql
CASE
    WHEN 条件1 THEN 结果1
    WHEN 条件2 THEN 结果2
    ELSE 默认结果
END
```

例如：

```sql
CASE
    WHEN supplier_type = '1' THEN '经销商'
    WHEN supplier_type = '2' THEN '分销商'
    WHEN supplier_type = '3' THEN '仓库'
    WHEN supplier_type IS NULL THEN '类型为空'
    ELSE '未知'
END
```

这种就更像平时写代码里的：

```text
if ...
else if ...
else ...
```

每一个 `WHEN` 后面都可以写完整条件。

比如判断金额区间：

```sql
CASE
    WHEN amount >= 10000 THEN '大额订单'
    WHEN amount >= 1000 THEN '普通订单'
    WHEN amount > 0 THEN '小额订单'
    WHEN amount = 0 THEN '零金额订单'
    WHEN amount IS NULL THEN '金额为空'
    ELSE '异常金额'
END
```

再比如多个字段一起判断：

```sql
CASE
    WHEN cooperation_status = '1' AND supplier_type = '1' THEN '合作经销商供货'
    WHEN cooperation_status = '1' AND supplier_type = '3' THEN '合作仓库供货'
    WHEN cooperation_status IS NULL THEN '合作状态为空'
    ELSE '其他情况'
END
```

所以简单记一下：

* 简单 CASE：适合一个字段和固定值做等值匹配
* 搜索式 CASE：适合写完整判断条件



## 为什么 `WHEN NULL` 不生效

重点来了。

前面的错误 SQL 是：

```sql
CASE t1.VC2SUPPLIER1TYPE
    WHEN NULL THEN ...
END
```

因为这是简单 CASE，所以它实际类似于：

```sql
t1.VC2SUPPLIER1TYPE = NULL
```

但 SQL 里判断空值，不能用 `=`。

也就是说下面这种写法是不对的：

```sql
字段 = NULL
```

正确写法应该是：

```sql
字段 IS NULL
```

这个地方和 Java、PHP、JS 里判断变量是否为空的感觉不太一样。SQL 里的 `NULL` 不是一个普通值，它表示未知。

所以：

```sql
NULL = NULL
```

也不会被当成 `TRUE`。

这就导致：

```sql
WHEN NULL THEN ...
```

看起来像是判断空值，实际上不会命中。

最后就会走到：

```sql
ELSE t1.VC2SUPPLIER1CODE
```

所以查询结果里供货方编码才还是原始 ID。



## 正确写法

这里要改成搜索式 CASE，把 `NULL` 判断写成 `IS NULL`。

供货方编码：

```sql
CASE
    WHEN t1.VC2SUPPLIER1TYPE = '1' THEN store_supplier_dealer.VC2ID
    WHEN t1.VC2SUPPLIER1TYPE = '2' THEN store_supplier_distributor.VC2CODE
    WHEN t1.VC2SUPPLIER1TYPE = '3' THEN store_supplier_warehouse.VC2WHSENUM
    WHEN t1.VC2SUPPLIER1TYPE IS NULL THEN COALESCE(
        (SELECT d2.VC2ID
           FROM MM_DEALER d2
          WHERE d2.ID = t1.VC2SUPPLIER1CODE),
        (SELECT dt2.VC2CODE
           FROM MM_DISTRIBUTOR dt2
          WHERE dt2.ID = t1.VC2SUPPLIER1CODE),
        (SELECT w2.VC2WHSENUM
           FROM MM_WAREHOUSE w2
          WHERE w2.ID = t1.VC2SUPPLIER1CODE),
        t1.VC2SUPPLIER1CODE
    )
    ELSE t1.VC2SUPPLIER1CODE
END AS 供货方编码
```

供货方名称：

```sql
CASE
    WHEN t1.VC2SUPPLIER1TYPE = '1' THEN store_supplier_dealer.VC2NAME
    WHEN t1.VC2SUPPLIER1TYPE = '2' THEN store_supplier_distributor.VC2NAME
    WHEN t1.VC2SUPPLIER1TYPE = '3' THEN store_supplier_warehouse.VC2WHSENAME
    WHEN t1.VC2SUPPLIER1TYPE IS NULL THEN COALESCE(
        (SELECT d2.VC2NAME
           FROM MM_DEALER d2
          WHERE d2.ID = t1.VC2SUPPLIER1CODE),
        (SELECT dt2.VC2NAME
           FROM MM_DISTRIBUTOR dt2
          WHERE dt2.ID = t1.VC2SUPPLIER1CODE),
        (SELECT w2.VC2WHSENAME
           FROM MM_WAREHOUSE w2
          WHERE w2.ID = t1.VC2SUPPLIER1CODE),
        t1.VC2SUPPLIER1NAME
    )
    ELSE t1.VC2SUPPLIER1NAME
END AS 供货方名称
```

这样逻辑就对了：

* 类型是 `1`，走经销商
* 类型是 `2`，走分销商
* 类型是 `3`，走仓库
* 类型是空，按 ID 去三个表里兜底查
* 都查不到，返回原字段



## COALESCE 这里是干啥的

顺便再记一下 `COALESCE`。

```sql
COALESCE(a, b, c, d)
```

意思是从左到右，取第一个非空的值。

比如：

```sql
COALESCE(dealer_code, distributor_code, warehouse_code, raw_code)
```

就可以理解为：

* `dealer_code` 有值，返回 `dealer_code`
* `dealer_code` 为空，再看 `distributor_code`
* `distributor_code` 为空，再看 `warehouse_code`
* 前面都为空，最后返回 `raw_code`

所以这次的逻辑里：

* `CASE` 是先判断供货方类型
* `COALESCE` 是在供货方类型为空的时候，按顺序兜底取值



## 还有几个顺手记录的点

### 1. CASE 是从上往下判断的

例如：

```sql
CASE
    WHEN amount >= 1000 THEN '大于等于1000'
    WHEN amount >= 500 THEN '大于等于500'
    ELSE '小于500'
END
```

如果 `amount = 1200`，第一个条件已经满足了，结果就是：

```text
大于等于1000
```

后面的就不会再判断。

所以范围判断的时候，顺序要注意。

下面这种就不太对：

```sql
CASE
    WHEN amount >= 500 THEN '大于等于500'
    WHEN amount >= 1000 THEN '大于等于1000'
    ELSE '小于500'
END
```

因为只要金额大于等于 500，就先命中了，后面的 `amount >= 1000` 基本就没机会了。



### 2. ELSE 不写的话，默认返回 NULL

例如：

```sql
CASE
    WHEN status = '1' THEN '合作'
END
```

如果 `status` 不是 `'1'`，结果就是 `NULL`。

所以业务查询里还是建议把 `ELSE` 写上，不然后面看结果的时候容易懵。

```sql
CASE
    WHEN status = '1' THEN '合作'
    WHEN status = '0' THEN '不合作'
    ELSE '未知状态'
END
```



### 3. NVL 和 CASE 不是一回事

如果只是空值替换，用 `NVL` 就行：

```sql
NVL(supplier_type, '类型为空')
```

但是如果是这种：

```text
类型是 1，取经销商编码
类型是 2，取分销商编码
类型是 3，取仓库编码
类型为空，再按多个表兜底查
```

这就不是简单的空值替换了，还是得用 `CASE`。



## 总结

这次问题本质就是把简单 CASE 和搜索式 CASE 混着用了。

错误点：

```sql
CASE 字段
    WHEN NULL THEN ...
END
```

这个不会按预期判断空值。

判断 `NULL` 要写：

```sql
字段 IS NULL
```

所以只要 `CASE` 里要判断 `NULL`、范围、多字段条件、`IN`、`LIKE` 这种，就直接写搜索式 CASE：

```sql
CASE
    WHEN 字段 IS NULL THEN ...
    WHEN 字段 = '1' THEN ...
    ELSE ...
END
```

简单 CASE 就留给这种固定值转换：

```sql
CASE 字段
    WHEN '1' THEN '经销商'
    WHEN '2' THEN '分销商'
    ELSE '未知'
END
```

以后看到 `WHEN NULL`，基本可以先怀疑这里写错了。

