---
title: 基于使用场景谈TestNG+JMockit+AssertJ的最佳实践
date: 2019-04-02 00:50:26
tags: UnitTest
categories: 工具
---

## 个人对单测的理解
1. 程序的结构只因架构设计变化而变化，方法访问权限在够用的情况下，越小越好，这些不能因为单测的影响而改变。
2. 单测也是代码的一部分，评估需求，排期都要算在内。
3. 交付的标准里应该包括至少90%的单测覆盖率。
3. 单测代码也是要被测试的，所以**突变测试(mutation testing)**（改程序逻辑，单测应该能用“Fail”检测到这个变更）也要抽样做。
<!--more-->
## 不说Feature，只谈场景
分别介绍TestNG/JMockit/AssertJ三个框架feature的文章汗牛充栋，在此不赘述，但是就像[《最佳实践有多重要？》](https://www.atatech.org/articles/96929)这篇文章里提到的，没有最佳实践，很多feature不知道什么情况下适用，慢慢也就忘记还有这样的feature了。


下面列出了一些单测中遇到的比较棘手的场景，和对应的最佳实践，如果有更好的做法，欢迎补充、更新。

---
### 场景0，private method，怎么单测？
一般有两种处理方式：

1. 反射后，`setAccessible(true)`，并执行
2. 跟随private的调用方一起单测

个人认为选择的策略是考虑private方法逻辑的独立性，独立性强，入参容易构建，选1，反之，选2。

### 场景1，void method，且逻辑stateful，怎么单测？
Java里的void method往往对应了procedure的概念，而不是function，没有可供assert的输出。如果观察逻辑是有状态的，且被private修饰，可以通过反射+`setAccessible(true)`的方式拿到对象的状态，再通过assert验证状态的正确性。

### 场景2，void method，且逻辑stateless，怎么单测？
有一种不太常见的情况，逻辑无状态，且method不输出结果，也不输出到对象状态，而是输出到框架和运行环境，几乎没有可以assert的内容。

e.g. ODPS的UDTF

```java
public void process(Object[] args) throws UDFException, IOException {
    for (int i = 0; i <10; i++) {
       forward(i, "abc");//forward内部实现会因运行环境不同而不同
    }
}
```

这种情况可以采用JMockit，将被调用的框架方法“劫持”，把框架方法的实现换成可以产生状态的逻辑，最后输出结果，并assert验证。伪代码如下，注意这里mock的方法并不是public，事实上，即使private也是可以的，这也是JMockit的抓手之一：
```java
List<String> list = Lists.newArrayList(xx,yy,zz);
Map<Object, Object> result = Maps.newHashMap();
ExpectedTimePositionExplode mock = new MockUp<ExpectedTimePositionExplode>(){
    @Mock
    protected void forward(Object... outs) throws UDFException {
        result.put(outs[0], outs[1]);
    }
}.getMockInstance();
mock.process(new Object[]{list});
assertThat(result).hasSize(3).containsValues(0,12,7);
```

### 场景3，逻辑里大量和运行环境的交互，构造环境很困难，怎么单测？
如果一段逻辑中，大量需要和运行环境交互，这时开发者一般为了避免麻烦，会mock大段逻辑，覆盖率会下降，不推荐。

推荐做法有三种：<br>


1. 基于行为mock：
```java
new Expectations() {
    {
        parquetReader.read();
        SimpleRecord simpleRecord = new SimpleRecord();
        simpleRecord.add("abc", "jared");
        result = simpleRecord;
    }
};
```

2. 基于对象mock：
```java
ExecutionContext mockExecutionContext = new MockUp<ExecutionContext>() {
    @Mock
    public void claimAlive() {}
}.getMockInstance();
```
3. 使用运行环境提供的local-implement，e.g. ODPS `SourceInputStream` 接口的实现，在运行时有专门的基于网络的实现，且不开源，单测时无法构造，且诸多方法都需要mock，mock时还需要了解每一个API的定义，很麻烦。但是随接口发布的还有一个本地实现 `LocalInputStream`，单测时可以利用上，个人理解这种本地实现应该就是为了调试方便而设计的。

### 场景4，逻辑有依赖关系，怎么在单测中体现出来？
代码使用了Template模式时，往往父类定义逻辑执行顺序，子类具体实现每一个方法。如下面三个方法执行顺序是setup(初始化) -> extract(正式业务) -> close(关闭资源)
```java
public abstract class Extractor {
  public abstract void setup(ExecutionContext ctx, InputStreamSet inputs, DataAttributes attributes);
  public abstract Record extract() throws IOException;
  public abstract void close();
}
```
对上面三个方法实现的单测，一般也需要按顺序依次进行，这时可以利用`@Test(dependsOnMethods = "setup")`在每一个单测方法上指定依赖的上游，上游完成下游执行；上游报错，下游ignore。

### 场景5，每次maven构建带单测过程漫长，怎么破？
构建时很慢可以采用多线程执行单测，以减少单测时间。

在testng.xml中可以设置thread-count和需要并行执行的单元（方法、类或suite）
```xml 
<suite name="Concurrency Suite" parallel="methods" thread-count="3" >
```

同时pom.xml中设置

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <suiteXmlFiles>
            <suiteXmlFile>src/test/resources/testng.xml</suiteXmlFile>
        </suiteXmlFiles>                   
    </configuration>
</plugin>
```

> 值得注意的是，即使在多线程单测环境下，`@Test(dependsOnMethods = "method1")`也是得到保证的，依赖关系不会乱，可以放心用。


### 场景6，验证线程安全性问题，怎么单测？
用`@Test(threadPoolSize = 3, invocationCount = 6, timeOut = 1000)`注解测试方法，可以构造并发执行环境，用于验证线程安全性问题。

e.g.
```java
int i = 0;
@Test(threadPoolSize = 10, invocationCount = 100, timeOut = 1000)
public void method1() {
   i++;
}
@Test(dependsOnMethods = "method4")
public void method2() {
    assertThat(i).isEqualTo(100);
}
```
运行method2，结果：
```
org.junit.ComparisonFailure: 
Expected :[100]
Actual   :[97]
```

### 场景7，多个边界测试数据测相同逻辑，写多个测试case代码冗长且测试数据不能复用，怎么破？
如果你写过这样的单测
```java
evaluate = distance.evaluate(7);
assertThat(evaluate).isGreaterThan(0);
evaluate = distanceBucket.evaluate(9);
assertThat(evaluate).isGreaterThan(0);
evaluate = distanceBucket.evaluate(5);
assertThat(evaluate).isGreaterThan(0);
```
那你可能需要考虑一下`@DataProvider`注解，帮你分离测试数据和逻辑，逻辑更清晰简洁，还可以实现测试数据的复用。
```java
@Test(dataProvider = "ppp")
public void testParameters(Integer p) {
    assertThat(p).isGreaterThan(0);
}

@DataProvider(name = "ppp")
public Object[] paramProvider() {
    return new Object[] {7, 9, 5};
}
```
