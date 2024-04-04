---
title: Dynamic-PB-Parser：搞定Protobuf数据动态解析
date: 2019-04-17 00:45:47
tags: pb-parser
categories: 开源产品
---

## 背景
### 核心痛点
使用ODPS(hive) SQL解析JSON数据的时候你可能用过get_json_object这个内置UDF，其灵活性给JSON数据解析带来了很多的便利。

但是对于Protobuf描述的数据，解析工作就麻烦太多了，即使解析一个字段，也要写一个完整的UDF(或UDTF)，并且上传各种protobuf的jar，然而这还没有完，如果新需求来了，即使仅改了一个字段的解析需求，也需要再重新开发一个新的UDF来完成。时间久了，项目里会有很多地方散落着各种Protobuf数据解析的逻辑，如果再有些公共逻辑改变，还要涉及到重构。维护工作繁重。
<!--more-->
### 本质原因
出现以上问题的本质原因实际上是我们把“我们想要A字段”和“我们如何得到A字段”两件事给耦合在一起了。按照函数式的编程思想，“控制”和“逻辑”要分开，并且互不干扰，这个问题里，“我们如何得到A字段”就是控制，而“我们想要A字段”就是逻辑。

### 破局
既然找到了耦合点，想办法解耦就是出路了。
首先，定义一套描述字段路径的语法规则，让它去表达“我们想要A字段”这件事；然后，开发一个可以解析这套语法的程序（类似编译器），去完成“我们如何得到A字段”这件事。这样就可以让两件事互相独立演进了。至此， `Dynamic-PB-Parser`也就呼之欲出了。


## `Dynamic-PB-Parser` 能做什么

一句话概括`Dynamic-PB-Parser`的功能：
>按一种简单的语法规则描述字段路径，并动态解析Protobuf数据中对应的字段值。

### Demo
```Java
DynamicPBParser parser = DynamicPBParser.newBuilder()  
.descFilePath("xxx/xxx.desc")  
.syntax("StandardSyntax")  
.build(); 
String name = parser.parse(content, 'me.lihongyu.bean.Person$name');  
BrandType brandType = parser.parse(content, 'me.lihongyu.bean.Person$cloth.brand.type');  
String email = parser.parse(parser.parse(content, 'me.lihongyu.bean.Person$proto_data'), 'me.lihongyu.bean.AddressBook$email');
;
```
## `Dynamic-PB-Parser` 怎么做的

总体的思路：动态拿到schema(也就是message的定义)，用schema解析pb的数据得到一个完整对象，沿着字段路径，不断循环得到下一级对象直到最后一级。一图胜千言：
![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/acda938c9e08972c18fb2217c0fab686.png)

### 实现过程中的一些细节
#### 可扩展的语法接口
在`Dynamic-PB-Parser`里定义了`Syntax`接口，其中定义了语法相关的解析逻辑，实现此接口，可以自定义一套自己的字段路径语法规则。默认提供的标准语法类似这样：`pkg1.XXX $ field1.field2[*].(pkg1.extension_field)[1]`，既支持数据，也支持用`()`表达的扩展字段。详细介绍见参考中的ATA文章。

#### 获取FieldValue
按FieldDescriptor获取FieldValue时，如果是扩展字段，需要在`UnknownFields`里找到字段值，并按不同字段类型选择不同的取值逻辑。

|字段值存储位置(in Field)|字段类型|取值逻辑|
|---|---|---|
|VarintList|ENUM|按ENUM定义的序号获取|
|VarintList|BOOLEAN|0代表FALSE、1代表TRUE|
|VarintList|INT|直接取VarintList[0]|
|VarintList|LONG|直接取VarintList[0]|
|Fixed32List|FLOAT|将用int表达的字节转为float：Float.intBitsToFloat|
|Fixed64List|DOUBLE|将用long表达的字节转为double：Double.longBitsToDouble|
|LengthDelimitedList|MESSAGE|DynamicMessage.parseFrom(LengthDelimitedList[0])|
|LengthDelimitedList|STRING|LengthDelimitedList[0].toStringUtf8()|
|LengthDelimitedList|BYTE_STRING|直接输出LengthDelimitedList[0]|


## `Dynamic-PB-Parser` 的性能优化

### 解析逻辑的两个阶段
`Dynamic-PB-Parser`将解析逻辑分为`load`和`parse`两个阶段：
1. `load`阶段完成schema的加载过程，其中包括了对扩展字段的schema处理
2. `parse`阶段完成字段路径的解析，和按路径逐步找到字段值的功能

### 通过缓存减少重复的parse逻辑
1. `descriptorCache`在`load`阶段初始化，缓存了descriptorName和descriptor的对应关系，在同一个JVM里只需要初始化一次即可
3. `extensionFieldCache`在`load`阶段初始化，缓存了desc文件里全部扩展字段名和扩展字段实例的对应关系。用于后续解析扩展字段时提取FieldDescriptor
4. `fieldPathCache`在`parse`阶段初始化，缓存了完整字段路径和字段拆分后结果的对应关系。避免每次都需要做split和对`()`表达的扩展字段路径的解析
5. `fieldNameCache`和`fieldIndexCache`在`parse`阶段初始化，分别缓存了field和fieldName（`abc[*]->abc`），以及field和fieldIndex（`abc[*]->*`）的对应关系，避免每次解析都要执行正则匹配。

## 写在最后
`Dynamic-PB-Parser`具体的实现细节还有很多，感兴趣的同学可以checkout Github repo。虽然不是什么高深的技术，但是背后也是对“控制和逻辑正交”的函数式编程思维的一次成功实践。
由于所在项目组的特殊性，这种用Protobuf描述的数据又非常多且普遍，用传统的protobuf的解析方式给开发人员的维护工作带来了很大痛苦，因此开发了`Dynamic-PB-Parser`这个工具，希望能帮助到更多和我有类似问题的开发同学。

## 参考
hive UDFJson:https://github.com/apache/hive/blob/master/ql/src/java/org/apache/hadoop/hive/ql/udf/UDFJson.java
github：https://github.com/lhyundeadsoul/pb-parser
欢迎提issue和MR

