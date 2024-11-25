---
title: 【知识补丁】 HIVE UDF开发 自定义UDF类成员的生效范围
date: 2022-07-26 15:32:02 +0800
categories: [HIVE]
tags: [知识补丁]
---



这里均是基于对自定义UDF函数（继承org.apache.hadoop.hive.ql.exec.UDF）使用，GenericUDF在另外的文档记录

### 参考文档：

- 【稀土掘金】UDF开发手册 - UDF：https://juejin.cn/post/6929325816836800520
- Hive中的UDF详解：https://www.cnblogs.com/data-magnifier/p/14167382.html
- Hive UDF加载耗时资源的一种解决思路：https://tech.huolala.cn/?p=210

## 自定义UDF类成员的生效范围

一个java类成员通常包含：
* 构造函数
* 静态变量
* 静态方法块
* 静态方法
* 成员变量
* 非静态代码块
* 成员方法

我们需要了解我们的自定义udf类，这些成员的发挥作用的时机和生命周期，来帮助我们开发udf功能。为了测试这些功能，我们构造了一个特殊的测试udf类:

- 包含静态和非静态的变量和代码方法块，和构造函数。
- 在执行的时候输出静态和非静态变量
- 因为静态和非静态变量分别是在静态和非静态代码方法块中赋值的随机数，可以通过hivesql调用udf的效果来确认类加载和对象创建的情况。

如下：

```java
package com.wb.search.logquality.mobile.query;

import org.apache.hadoop.hive.ql.exec.UDF;

import java.util.Random;

public class TestUDF extends UDF {
    // * 构造函数
    public TestUDF() {
        System.out.println("构造函数执行:" + this);
    }

    //* 静态变量
    static int staticVal;

    //* 静态代码块
    static {
        System.out.println("静态代码块执行");
        Random r = new Random();
        staticVal = r.nextInt(100);
    }

    //* 静态方法
    static public int TestStatic() {
        return 100;
    }

    //* 成员变量
    int val;

    //* 非静态代码块
    {
        System.out.println("非静态代码块执行:" + this);
        Random r = new Random();
        val = r.nextInt(100);
    }

    //* 成员方法
    public String evaluate(String s) {
        return s + "\t|静态变量：" + staticVal + "\t|成员变量：" + val;
    }

    public static void main(String[] args) {
        TestUDF testUDF = new TestUDF();
        System.out.println(testUDF.evaluate("test"));
//        静态代码块执行
//        非静态代码块执行:com.wb.search.logquality.mobile.query.TestUDF@5cb0d902
//        构造函数执行:com.wb.search.logquality.mobile.query.TestUDF@5cb0d902
//        test	|静态变量：83	|成员变量：63
//        test2	|静态变量：83	|成员变量：63
//        非静态代码块执行:com.wb.search.logquality.mobile.query.TestUDF@46fbb2c1
//        构造函数执行:com.wb.search.logquality.mobile.query.TestUDF@46fbb2c1
//        test3	|静态变量：83	|成员变量：20
    }
}

```

通过输出我们可以看到执行顺序：静态代码块->非静态代码块->构造函数，如果有新的对象构建了，会输出新的随机的成员变量。

### 1. hive sql调用情况

#### 1.1 hive shell

接下来我们打包上传集群，观察下在多map或reduce下的执行情况，首先是直接在**hive shell**中执行：

```sql
hive> create temporary function test_udf as 'com.search.logquality.mobile.query.TestUDF';
静态代码块执行
OK
Time taken: 0.007 seconds
hive> select test_udf(1) from wbsearch.ods_mobile_query where dt='20220720' limit 10;
非静态代码块执行:com.wb.search.logquality.mobile.query.TestUDF@60652518
构造函数执行:com.wb.search.logquality.mobile.query.TestUDF@60652518
非静态代码块执行:com.wb.search.logquality.mobile.query.TestUDF@404009df
构造函数执行:com.wb.search.logquality.mobile.query.TestUDF@404009df
非静态代码块执行:com.wb.search.logquality.mobile.query.TestUDF@75e4fe3c
构造函数执行:com.wb.search.logquality.mobile.query.TestUDF@75e4fe3c
非静态代码块执行:com.wb.search.logquality.mobile.query.TestUDF@7bda01da
构造函数执行:com.wb.search.logquality.mobile.query.TestUDF@7bda01da
OK
1	|静态变量：35	|成员变量：11
1	|静态变量：35	|成员变量：11
1	|静态变量：35	|成员变量：11
1	|静态变量：35	|成员变量：11
1	|静态变量：35	|成员变量：11
1	|静态变量：35	|成员变量：11
1	|静态变量：35	|成员变量：11
1	|静态变量：35	|成员变量：11
1	|静态变量：35	|成员变量：11
1	|静态变量：35	|成员变量：11
Time taken: 1.611 seconds, Fetched: 10 row(s)

```

通过执行可知：

- 静态代码块在创建临时方法的时候调用
- 非静态代码块在执行sql的时候执行，而且执行了三次，原因后面说明。
- 多行日志多次调用udf，对象没有重复创建，一直使用最后一次创建的udf对象（成员变量都是11）

#### 1.2 hive sql on yarn

我们执行一个sql,把结果写到hdfs:

```sql
add jar viewfs://xxx/packages/udf/SearchUDF.jar;
create temporary function test_udf as 'com.search.logquality.mobile.query.TestUDF';
insert overwrite directory 'viewfs://xxx/tmp/test_data2'
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES ('field.delim'='\t','serialization.null.format'='')

select test_udf(new_user_level) from wbsearch.ods_f_level_property_field_by_field where dt='20220720';
```

经过检查输出结果：

```shell
[wb_search_bas@10 test_data]$ head -n 3 *
==> 000000_0 <==
-1	|静态变量：32	|成员变量：26
5	|静态变量：32	|成员变量：26
-1	|静态变量：32	|成员变量：26

==> 000001_0 <==
-1	|静态变量：66	|成员变量：87
-1	|静态变量：66	|成员变量：87
-1	|静态变量：66	|成员变量：87

==> 000002_0 <==
5	|静态变量：70	|成员变量：57
5	|静态变量：70	|成员变量：57
5	|静态变量：70	|成员变量：57

==> 000003_0 <==
-1	|静态变量：12	|成员变量：73
-1	|静态变量：12	|成员变量：73
-1	|静态变量：12	|成员变量：73
```

这个表的20220720分区有四个文件，正好对应输出的4个文件，我们看到四个输出文件的静态变量和成员变量的值一致。**说明每个map都加载了这个类，并仅创建了一个udf类对象，重复调用evaluate实现的调用udf功能。**
