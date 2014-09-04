---
layout: post
title: "Writing a Hive Generic UDF"
date: 2014-09-05 05:40:22 +0800
comments: true
categories: hive udf genericudf
---

## 业务背景

公司的数据分析业务 ETL 过程中，原始数据经过数据清洗之后，统一转换成自定义的 protobuf `CustomEventMessage` 格式，用户自定义事件 ( 多个KV对的格式 ) 保存在 `event_args` 字段中。最终 `CustomEventMessage` 数据将导入 `hive` 中，在没有 `Protobuf Serde` 之前，这需要先将数据重新转换为文本格式。
最后，在 `hive` 的建表语句中 `event_args` 字段声明为 `MAP` 类型：

```sql
    event_args MAP<INT, INT>
or
    event_args MAP<STRING, STRING>
```

举例来说，某个 `event_args` 字段带有两个自定义参数 `money` 和 `paytype`，转换后的文本内容为：（ Key 1 和 3 是预定的 ）

```sql
    1:200|3:1
or
    money:200|paytype:1
```

在后续 hive 查询脚本中引用该字段提取其中特定的事件（ 比如支付类型 `paytype` ）很简单且直观：

```sql
    event_args[3]
or
    event_args['paytype']
```

### 存在的问题及对策

多了转文本这一步骤，维护起来太繁琐，因此开发了 **ProtobufSerde** ，让 `hive` 能直接支持存取 `protobuf` 格式的数据，这样 `CustomEventMessage` 入库 hive 将会方便很多。

当然，现在的 hive 程序也需要做一些相应的调整，比如上述 `event_args` 字段的读取就麻烦得多了。现在的建表语句变成如下形式（直接对应 proto）：

```sql
event_args ARRAY< STRUCT<key_int:BIGINT,key_str:STRING,value_int:BIGINT,value_str:STRING> >
```

`select` 得到的数据格式将是类似这样的：

```sql
[{"key_int":1,"key_str":"money","value_int":200,"value_str":"200"},{"key_int":3,"key_str":"paytype","value_int":1,"value_str":"1"}]
```

在这里例子中，为了得到 `paytype`，我们可以在查询语句中这样书写：

```sql
SELECT event_args[1].value_int FROM s_CustomEventMessage WHERE pyearmonth='201403' and pday='07' and event_id=30;
```

### 引入的新问题

看起来依旧方便的这种写法，基于 `2` 个前提：

> *  `event_args` 中有 `paytype` 事件
> * `paytype` 自定义事件在 `event_args` 中是第 `2` 个 item（ index 1 ）

遗憾的是，这两个条件都没有办法满足。因此现在需要通过 遍历整个 `event_args` 的内容，找出我们所关心的事件。
OK，很明显，我们需要 `UDF` 来完成这一任务。


----------

## 如何编写 Hive GenericUDF

为编写 UDF，Hive 中提供了两套不同的接口，分别是：
> * Simple API - `org.apache.hadoop.hive.ql.exec.UDF`
> * Complex API - `org.apache.hadoop.hive.ql.udf.generic.GenericUDF`

`UDF` 接口非常容易编写，但其局限性在于它只适用于函数输入参数和返回值都是 `Java原生类型` 的场景，这里我们的输入参数类型是 `ARRAY< STRUCT<key_int:BIGINT,key_str:STRING,value_int:BIGINT,value_str:STRING> >`，返回值是 `BIGINT` 或者 `STRING` 类型，因此 `UDF` 接口满足不了需求。

GenericUDF 是 Hive UDFs 中的 **瑞士军刀**，适用处理 Hive 的复杂类型如 `MAP`、`ARRAY`、`STRUCT` 以及在此之上的各种嵌套类型。下面我们就使用它来完成这一任务。

### 几点准备知识：
> * UDF 函数通过 `ObjectInspector` 来访问传入的参数，因此 `ObjectInspector` 的概念很重要
（对于 ObjectInspector 不太懂，等了解学习以后单独写个文档介绍）
> * GenericUDFs 中使用的很多函数，特别是 `ObjectInspector` 的众多子类函数，返回 `Object` 类型，这意味着：
（1） 编译器无法帮助我们做类型检查
（2） 很多时候我们不知道拿到手的对象的实际类型（hadoop 的 tasklogs 是很有用的工具，可以借助它来调试）

所有的自定义 `GenericUDFs` 继承自 `GenericUDF` 类，因此需要实现以下三个函数：

```java
public ObjectInspector initialize(ObjectInspector[] arguments)
public Object evaluate(DeferredObject[] arguments)
public String getDisplayString(String[] children)
```
### 1） initialize()函数

initialize() 函数在 UDF 第一次调用时执行，主要完成四件事：
> * 检查输入参数类型（在我们的例子中，即 `array<struct<key_int:bigint,key_str:string,value_int:bigint,value_str:string>>` 和 `string`）
> * 设置并返回一个跟函数返回值匹配的 `ObjectInspector` 对象
> * 把输入参数中用到的各种 `ObjectInspectors` 保存到全局变量中
> * 设置返回值

### 2）evaluate() 函数

evaluate() 函数接收输入实参，完成计算后返回结果。如前所述，我们通过各种 `ObjectInspectors` 来访问传入的参数，`ObjectInspectors` 是由我们在 initialize() 函数中保存起来的。我们需要知道：
> * array<> 在 Java 中用 ArrayList<> 来表示
> * struct<> 在 Java 中用 Object[] 来表示
> * 使用 hadoop 的 Writable 和 Text 类而非 Java 原生类型

### 3） getDisplayString() 函数

不那么重要，我们只需返回一个字符串，表示该 UDF 是干什么的就行了，在我们使用 `EXPLAIN` 调试 `hql` 语句时，该信息会被输出。


----------


## 代码实现

代码中已经有了详尽的注释来解释每一个步骤，可以参考着上面的内容来理解。不再详述。

```java
package com.mjyun.hive.udf;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.IntWritable;

import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.exec.UDFArgumentLengthException;
import org.apache.hadoop.hive.ql.exec.UDFArgumentTypeException;
import org.apache.hadoop.hive.ql.metadata.HiveException;

import org.apache.hadoop.hive.ql.udf.generic.GenericUDF;

import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector.Category;
import org.apache.hadoop.hive.serde2.objectinspector.ListObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.StructObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.StandardListObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;

import org.apache.hadoop.hive.serde2.objectinspector.StructField;

import org.apache.hadoop.hive.serde2.objectinspector.PrimitiveObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;

import org.apache.hadoop.hive.serde2.objectinspector.primitive.StringObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.LongObjectInspector;

import org.apache.hadoop.hive.serde2.lazy.LazyString;
import org.apache.hadoop.hive.serde2.lazy.LazyLong;

import java.util.ArrayList;


public class UDFGetValueFromEventArgsStr extends GenericUDF {

    // Global variables that inspect the input.
    // These are set up during the initialize() call, and are then used during the calls to evaluate()
    private ListObjectInspector   listOI;                   // ObjectInspector for the list (array<>)
    private StructObjectInspector structOI;                 // ObjectInspector for the struct<>
    private ObjectInspector       kiOI, ksOI, viOI, vsOI;   // ObjectInspector for the elements of the struct<>: key_int, key_str, value_int, value_str
    private StringObjectInspector targetOI;                 // ObjectInspector for the second arg that we match with target field

    public String getDisplayString ( String []arg0 ) {
        return "getValueFromEventArgsStr";
    }

    // this is like the evaluate method of the simple API. 
    // It takes the actual arguments and returns the result.
    public ObjectInspector initialize ( ObjectInspector[] arguments ) throws UDFArgumentException {

        if ( arguments.length != 2 ) {
            throw new UDFArgumentTypeException(0, "getValueFromEventArgsStr only takes 2 arguments: List<Struct<bigint,string,bigint,string>>, String");
        }

        // 1. Check we received the right object types.
        ObjectInspector a = arguments[0];
        ObjectInspector b = arguments[1];
        if ( !( a instanceof ListObjectInspector ) ) {
            throw new UDFArgumentTypeException(0, "The first argument must be a Array<Struct>, "
                    + " but " + a.getTypeName() + " is found");
        }
        if ( !( b instanceof StringObjectInspector ) ) {
            throw new UDFArgumentTypeException(0, "The second argument must be a String, "
                    + " but " + b.getTypeName() + " is found");
        }

        // 2. Get the object inspector for the list(array) elements; this should be a StructObjectInspector
        // Then check that the struct has the correct fields.
        // Also, store the ObjectInspectors for use later in the evaluate() method
        listOI   = (ListObjectInspector)a;
        targetOI = (StringObjectInspector)b;
        structOI = (StructObjectInspector)listOI.getListElementObjectInspector();

        if ( structOI.getAllStructFieldRefs().size() != 4 ) {
            throw new UDFArgumentTypeException(0, "Incorrect number of fields in the struct. "
                    + "The first argument of type Array<Struct> must contains 4 fields, "
                    + " but " + structOI.getAllStructFieldRefs().size() + " fields is found");
        }

        StructField key_int   = structOI.getStructFieldRef("key_int");
        StructField key_str   = structOI.getStructFieldRef("key_str");
        StructField value_int = structOI.getStructFieldRef("value_int");
        StructField value_str = structOI.getStructFieldRef("value_str");

        if ( null == key_int )
            throw new UDFArgumentTypeException(0, "No \"key_int\" field in input structure " + arguments[0].getTypeName());
        if ( null == key_str )
            throw new UDFArgumentTypeException(0, "No \"key_str\" field in input structure " + arguments[0].getTypeName());
        if ( null == value_int )
            throw new UDFArgumentTypeException(0, "No \"value_int\" field in input structure " + arguments[0].getTypeName());
        if ( null == value_str )
            throw new UDFArgumentTypeException(0, "No \"value_str\" field in input structure " + arguments[0].getTypeName());

        kiOI = key_int.getFieldObjectInspector();
        ksOI = key_str.getFieldObjectInspector();
        viOI = value_int.getFieldObjectInspector();
        vsOI = value_str.getFieldObjectInspector();

        if ( kiOI.getCategory() != ObjectInspector.Category.PRIMITIVE )
            throw new UDFArgumentTypeException(0, "Is input primitive? key_int field must be a bigint, but " + kiOI.getTypeName() + " is found");
        if ( ksOI.getCategory() != ObjectInspector.Category.PRIMITIVE )
            throw new UDFArgumentTypeException(0, "Is input primitive? key_str field must be a string, but " + ksOI.getTypeName() + " is found");
        if ( viOI.getCategory() != ObjectInspector.Category.PRIMITIVE )
            throw new UDFArgumentTypeException(0, "Is input primitive? value_int field must be a bigint, but " + viOI.getTypeName() + " is found");
        if ( vsOI.getCategory() != ObjectInspector.Category.PRIMITIVE )
            throw new UDFArgumentTypeException(0, "Is input primitive? value_str field must be a string, but " + vsOI.getTypeName() + " is found");

        if ( ((PrimitiveObjectInspector)kiOI).getPrimitiveCategory() != PrimitiveObjectInspector.PrimitiveCategory.LONG )
            throw new UDFArgumentTypeException(0, "Is input correct primitive? key_int field must be a bigint, but " + kiOI.getTypeName() + " is found");
        if ( ((PrimitiveObjectInspector)ksOI).getPrimitiveCategory() != PrimitiveObjectInspector.PrimitiveCategory.STRING )
            throw new UDFArgumentTypeException(0, "Is input correct primitive? key_str field must be a string, but " + ksOI.getTypeName() + " is found");
        if ( ((PrimitiveObjectInspector)viOI).getPrimitiveCategory() != PrimitiveObjectInspector.PrimitiveCategory.LONG )
            throw new UDFArgumentTypeException(0, "Is input correct primitive? value_int field must be a bigint, but " + viOI.getTypeName() + " is found");
        if ( ((PrimitiveObjectInspector)vsOI).getPrimitiveCategory() != PrimitiveObjectInspector.PrimitiveCategory.STRING )
            throw new UDFArgumentTypeException(0, "Is input correct primitive? value_str field must be a bigint, but " + vsOI.getTypeName() + " is found");
        
        // 3. the return type of our function is a bigint, so we provide the correct object inspector
        return PrimitiveObjectInspectorFactory.javaLongObjectInspector;
    }

    // The evaluate() method. The input is passed in as an array of DeferredObjects, so that
    // computation is not wasted on deserializing them if they're not actually used
    public Object evaluate( DeferredObject[] arguments ) throws HiveException {

        // Should take 2 arguments
        if ( arguments.length != 2 )
            return null;

        if ( null == arguments[0].get() )
            return null;

        // Iterate over the elements of the input array
        // If a key_str of arguments[1] if found, return key_int
        // Else return null
        int nelements = listOI.getListLength(arguments[0].get()), val_int;
        for ( int i = 0; i < nelements; i++ ) {
            Text tt = (Text)(structOI.getStructFieldData( listOI.getListElement(arguments[0].get(), i), structOI.getStructFieldRef("key_str") ));
            String t = tt.toString();
            // String skey_str = targetOI.getPrimitiveJavaObject(LSkey_str);
            String target = targetOI.getPrimitiveJavaObject(arguments[1].get());
            if ( t.equals(target) ) {
                Long ll = ((LongWritable)(structOI.getStructFieldData( listOI.getListElement(arguments[0].get(), i), structOI.getStructFieldRef("value_int") ))).get();
                return ll;
            }
        }

        return null;
    }    

}
```


----------

## 小结

本文粗浅地介绍了一个处理 `hive` `array<struct<>>` 数据类型的 `GenericUDF` 应该如何编写，大概就是这些了，第一次接触，略作记录，仅供参考。


----------


## 参考 

 1. [官方GenericUDF API][1]
 2. [HivePlugins][2]
 3. [User-Defined Functions][3]
 4. [Structured data in Hive: a generic UDF to sort arrays of structs][4]
 5. [Hadoop Hive UDF Tutorial - Extending Hive with Custom Functions][5]
 6. [Writing a Hive Generic UDF][6]


  [1]: http://hive.apache.org/javadocs/r0.10.0/api/org/apache/hadoop/hive/ql/udf/generic/GenericUDF.html
  [2]: https://cwiki.apache.org/confluence/display/Hive/HivePlugins
  [3]: https://www.inkling.com/read/hadoop-definitive-guide-tom-white-3rd/chapter-12/ch12-section-08
  [4]: http://www.congiu.com/structured-data-in-hive-a-generic-udf-to-sort-arrays-of-structs/
  [5]: http://blog.matthewrathbone.com/2013/08/10/guide-to-writing-hive-udfs.html
  [6]: http://www.baynote.com/2012/11/a-word-from-the-engineers/
