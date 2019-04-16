---
title: dynamic-pb-parser 介绍
date: 2019-04-17 00:45:47
tags: pb-parser
categories: 开源产品
---
# `dynamic-pb-parser` 介绍

`dynamic-pb-parser`提供类似hive的`get_json_object`的功能，可以动态解析用Protobuf描述的数据。

## Demo
```java
DynamicPBParser.parse(content, 'geo.log.LogRecord$logMetaData.level');
DynamicPBParser.parse(content, 'geo.log.LogRecord$logMetaData.serverName');
DynamicPBParser.parse(content, 'geo.log.LogRecord$protobuf.extensionNames');
DynamicPBParser.parse(content, 'geo.log.LogRecord$protobuf.data');
DynamicPBParser.parse(DynamicPBParser.parse(content, 'geo.log.LogRecord$protobuf.data'), 'geo.log.RequestLogRecord$metaData.requestTime');
DynamicPBParser.parse(DynamicPBParser.parse(content, 'geo.log.LogRecord$protobuf.data'), 'geo.log.RequestLogRecord$metaData.appleRpcHeader.appId');
```
<!--more-->
## 使用方法

>因为Protobuf序列化的数据不能自解释，所以需要使用此函数的同学自行编译自己的proto文件为desc文件(命令见下文)，并将desc文件路径传入

### 接入方式
基于以上原因，此UDF提供两种接入方式

1. 在[这里](https://market.dw.alibaba-inc.com/#/detail/9920)申请公共UDF使用权限，但是使用时需要写明`access_id, access_key`
2. 使用jar包自行创建UDF，步骤多，不能自动更新版本，但是不需要写`access_id, access_key`

1. 使用protoc命令将proto文件转换为desc文件(desc文件名可自定义)：
    ```
    protoc --include_imports -I. -oall.desc *.proto
    ```
3. ```DynamicPBParser.parse(content, 'geo.log.LogRecord$protobuf.extensionNames');```

### 出参、入参和语法

6. `DynamicPBParser.parse`有两个入参：
	7. 用Base64编码后的pb数据
	8. 需要解析的字段路径
7. 字段路径语法：
	8. 使用`$`符号分隔类名和字段名
	8. 嵌套对象的格式：`package_name.message_name$field1_name.field2_name`
	9. 嵌套数组的格式：`package_name.message_name$field1_name[*].field2_name[0]`，其中`field1_name[*]`也可简写为`field1_name`
	10. 扩展字段
        10. 对于message A 扩展字段x 定义在message B里的情况，解析x，可以把A的数据当作B来看，例：
        
        ```
        package a.b;
        message A {
            extensions 100 to 199;
        }      
        
        package c.d;
        message B {
            extend A {
                optional int32 x = 100;
            }
        }
        
        data=Base64(A);
        result = get_pb_object(data, 'c.d.B$x'); 
        ```
        
        11. 对于message A 扩展字段x 未定义在某message里的情况，而是直接定义在package下的情况，解析x，需要写明确完整扩展字段路径，并用英文小括号`()`括起来，例：
        
        ```
        package a.b;
        message A {
            extensions 100 to 199;
        }      
        
        package c.d;
        extend A {
            optional int32 x = 100;//protobuf认为此字段的完整路径是`c.d.x`
        }
        
        data=Base64(A);
        result = get_pb_object(data, "a.b.A$(c.d.x)");
        ``` 
10. 出参：
	11. 永远是string类型
	12. 如果返回的是object
		13. 会返回用Base64编码后的object PB序列化结果，类似`"CggKBG5pa2UQARC2YA=="`
		13. 如果是Byte数组，会返回Base64编码后的字符串
		14. 其它情况返回toString()的结果
	13. 如果返回的是数组类型，会返回`[xxx,xxx,xxx]`
		14. 如果元素是数字类型，会返回数字本身，如`[1,2,3]`
		15. 其中当元素是object类型时，同2

# Q&A
1. 发生异常`java.lang.IllegalArgumentException: XXX.XXX can not be found in any description file! Please check out if it exist.`，请检查desc文件中是否含有XXX.XXX的message定义，或仔细核对是否`package_name`和`message_name`有笔误
2. 除问题1描述的场景外，其余错误统一返回null。如字段找不到、对象解析失败等

# Todo List

- [x] support extension field 
- [x] support array type
- [x] exception or null?
- [x] support proto importing  
- [x] optimize the performance

