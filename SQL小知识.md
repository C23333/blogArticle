```
title: MySQL字符串拆分，实现split（explode）功能
date: 2022-12-02 21:07:29
tags: 
  - MySQL
  - 字符串分割
keywords: MySQL
categories: 实操
```



### MySQL字符串拆分，实现split（explode）功能

> 更详细讲解请参考<a href="https://blog.csdn.net/pjymyself/article/details/81668157">此篇文章</a>，以下为简化后的个人理解版

* 需求场景：需要统计每条数据中stories单独作为一列查出来

![image-20221229174617439](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202212291746616.png)

* 要解决的点主要有两个，一个是可能存在坏数据（比如逗号出现在了第一位或最后一位），另一个是如何以`,`分割

  * 坏数据：通过MySQL的TRIM()函数处理

    ```sql
    TRIM(BOTH ',' FROM `stories`)
    ```

  * 以`,`分割

    * 首先通过REPLACE()函数将`,`去除，再与去除前的stories字段值的LENGTH()相减，可以得出一共有多少个`,`（记得首先去除两侧的`,`）

      ![image-20221229190736504](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202212291907545.png)

      ![image-20221229190756409](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202212291907439.png)

    * 利用SUBSTRING_INDEX()函数，参数传入正值，获取第n个`,`为分隔符之前的所有字符

      ![image-20221229191017266](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202212291910299.png)

    * 通过上边可以想到，只要能遍历当前条数据中stories字段含有的`,`数量 次，就可以分别取到第一个、第二个、第三个...`,`前的所有字符了，再对SUBSTRING_INDEX()的结果进行一次SUBSTRING_INDEX()操作，这次参数传入负值，这里传入`-1`，即取最后一个`,`之后的所有字符。即实现了每次获取第 n 和 n + 1 个 `,`之间的字符

    * 这里的 n 需要递增，这里使用MySQL库的helo_topic_id来作为变量，因为help_topic_id是自增的，当然，也可以用其他表的自增字段辅助（要确保连续自增，以实现类似 n++ 的操作）

  * 最终实现代码：

    ```sql
    SELECT
      id,
    	SUBSTRING_INDEX( SUBSTRING_INDEX( stories, ',', ht.help_topic_id), ',', -1 ) AS roleId
    FROM
    	zt_release
    	INNER JOIN mysql.help_topic ht ON ht.help_topic_id <= ( LENGTH( stories ) - LENGTH( REPLACE ( stories, ',', '' )) + 1 ) 
    ```

    ![image-20221229191745826](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202212291917891.png)

