---
layout: post
title:  "代码质量保障方案——方法和工具"
author: "见欢"
categories: 测试开发
description: 如何保障系统代码质量，这是一篇比较全面的总结文章。
keywords: 测试工程师, 代码质量
---

<a name="df368884"></a>
## 前言
  代码从编写-测试-发布，经历了一个完整的生命周期，纯粹的黑盒测试，仅仅只能涵盖业务逻辑，对非业务和异常流程往往很难覆盖全面。本文将针对java代码探讨代码质量保障的各种方案和工具，从而能形成一定的认知和方法论。

<a name="b24e43cb"></a>
## 一、代码规约
  代码规约其实就是代码在编写过程，还没实际运行到时候进行一定的规范检查，这个可以使用工具或人工进行。<br />  静态代码分析是指无需运行被测代码，仅通过分析或检查源程序的语法、结构、过程、接口等来检查程序的正确性，找出代码隐藏的错误和缺陷，如参数不匹配，有歧义的嵌套语句，错误的递归，非法计算，可能出现的空指针引用等等。<br />  在软件开发过程中，静态代码分析往往先于动态测试之前进行，同时也可以作为制定动态测试用例的参考。统计证明，在整个软件开发生命周期中，30% 至 70% 的代码逻辑设计和编码缺陷是可以通过静态代码分析来发现和修复的。<br />  静态代码分析工具的优势在于：能够快速定位代码隐藏错误和缺陷，提高软件可靠性并节省软件开发和测试成本。<br /><br />
<a name="b3b55733"></a>
#### Java 静态代码分析理论基础和主要技术
* 缺陷模式匹配：缺陷模式匹配事先从代码分析经验中收集足够多的共性缺陷模式，将待分析代码与已有的共性缺陷模式进行模式匹配，从而完成软件的安全分析。这种方式的优点是简单方便，但是要求内置足够多缺陷模式，且容易产生误报。
* 类型推断：类型推断技术是指通过对代码中运算对象类型进行推理，从而保证代码中每条语句都针对正确的类型执行。这种技术首先将预定义一套类型机制，包括类型等价、类型包含等推理规则，而后基于这一规则进行推理计算。类型推断可以检查代码中的类型错误，简单，高效，适合代码缺陷的快速检测。
* 模型检查：模型检验建立于有限状态自动机的概念基础之上，这一理论将被分析代码抽象为一个自动机系统，并且假设该系统是有限状态的、或者是可以通过抽象归结为有限状态。模型检验过程中，首先将被分析代码中的每条语句产生的影响抽象为一个有限状态自动机的一个状态，而后通过分析有限状态机从而达到代码分析的目的。模型检验主要适合检验程序并发等时序特性，但是对于数据值域数据类型等方面作用较弱。
* 数据流分析：数据流分析也是一种软件验证技术，这种技术通过收集代码中引用到的变量信息，从而分析变量在程序中的赋值、引用以及传递等情况。对数据流进行分析可以确定变量的定义以及在代码中被引用的情况，同时还能够检查代码数据流异常，如引用在前赋值在后、只赋值无引用等。数据流分析主要适合检验程序中的数据域特性。

  在写代码的时候就开始规范代码的编写方式能够很好的减少程序的bug，常用的工具有[checkstyle](http://checkstyle.sourceforge.net/)、[pmd](https://pmd.github.io/pmd-6.12.0/)、[findbugs](http://findbugs.sourceforge.net/)、macker、jtest。
<a name="checkstyle"></a>
#### checkstyle
  Checkstyle是一个源代码分析器用来帮助程序员编写Java代码,使大家能遵循一个编码标准。它按照一定的规范自动检查Java代码。
<a name="pmd"></a>
#### pmd
  PMD是一个源代码分析器。它发现常见的编程缺陷如未使用的变量,空catch块,不必要的对象创建,等等。它支持Java、JavaScript、Apache Velocity,XML,XSL等。
<a name="findbugs"></a>
#### findbugs
  FindBugs 是一个静态分析工具，它检查类或者 JAR 文件，将字节码与一组缺陷模式进行对比以发现可能的问题。有了静态分析工具，就可以在不实际运行程序的情况对软件进行分析。

这里主要介绍以上三个工具并做以下对比，macker和jtest请自行查阅相关文档。

| **Java 静态分析工具** | **分析对象** | **应用技术** | **IDE插件** | **是否支持命令行执行** | **是否开源** |
| --- | --- | --- | --- | --- | --- |
| checkstyle | Java 源文件 | 缺陷模式匹配 | eclipse<br />idea | 是 | 是 |
| pmd | Java 源文件 | 缺陷模式匹配 | eclipse<br />idea | 是 | 是，支持二次开发，推荐阿里开源代码规约项目[pmd-p3c](https://github.com/alibaba/p3c) |
| findbugs | 字节码 | 缺陷模式匹配<br />数据流分析 | eclipse<br />idea | 是 | 是，支持二次开发 |

<a name="00914c42"></a>
## 二、代码review
<a name="34150b53"></a>
#### codereview的三个切入点
  如果说代码规约可以使用工具，那代码review就基本只能通过人来进行了，严格意义来讲，代码规约是代码review的一部分，只是执行者是工具，可以把代码review从三个方面进行切入：
1. 常见代码问题

常见的潜在代码问题是当前直接会导致BUG、故障或者产品功能不能正常工作的类别
1. 可维护性问题

在当前业务变更范围内通常不会导致BUG、故障，却会在日后埋下地雷，引发BUG、故障、维护成本大幅增加
1. 更难发现的错误

复杂并发场景下的有一定技术难度的、需要丰富开发与设计经验才能看出来的错误<br />  以上三个方面的具体内容，请看下方的脑图，这里不展开描述，可以看到的是代码规约可以帮助我们规避1、2中的大部分问题。<br />![codereview.png](http://assets.processon.com/chart_image/5c828382e4b0c996d35c2505.png)

<a name="8e491e81"></a>
## 三、白盒测试
  代码规范和review没有关注太多的业务逻辑，关注业务逻辑并验证其正确性主要依靠白盒测试，白盒测试为后续的持续集成提供基础。根据代码的层次，白盒测试可以做dao层、service层、controller层。
<a name="4fabce43"></a>
### 测试分层

![测试金字塔.png](http://assets.processon.com/chart_image/5c86698de4b09a16b99e07e5.png)<br />  按照以上的测试分层，白盒测试基本上属于下方的单元测试和集成测试范畴。
<a name="634efc00"></a>
### 单元测试框架及差异
  junit是JAVA的官方单元测试框架，目前稳定且广泛使用的是4.12版本，这两年推出了下一代独立版本junit5。而testng，被人们誉为下一代单元测试框架，下面主要看testng包含哪些junit（这里主要指junit4）没有的功能。<br />![单元测试框架及差异.png](http://assets.processon.com/chart_image/5c870b00e4b01e76977a9cfa.png)<br />
<a name="9b942096"></a>
#### 注解支持
Junit4和TestNG的注解对比：

| 特性 | JUnit4 | TestNG |
| :--- | :--- | :--- |
| 测试注解 | @Test | @Test |
| 在测试套件执行之前执行 | – | @BeforeSuite |
| 在测试套件执行之后执行 | – | @AfterSuite |
| 在测试之前执行 | – | @BeforeTest |
| 在测试之后执行 | – | @AfterTest |
| 在测试组执行之前执行 | – | @BeforeGroups |
| 在测试组执行之后执行 | – | @AfterGroups |
| 在测试类执行之前执行 | @BeforeClass | @BeforeClass |
| 在测试类执行之后执行 | @AfterClass | @AfterClass |
| 在测试方法执行之前执行 | @Before | @BeforeMethod |
| 在测试方法执行之后执行 | @After | @AfterMethod |
| 忽略测试 | @ignore | @Test(enbale=false) |
| 预期异常 | @Test(expected = Exception.class) | @Test(expectedExceptions = Exception.class) |
| 超时 | @Test(timeout = 1000) | @Test(timeout = 1000) |

<a name="bd16a34a"></a>
#### 依赖测试
  依赖测试是指测试的方法是有依赖的，在执行的测试之前需要执行的另一测试。如果依赖的测试出现错误，所有的子测试都被忽略，且不会被标记为失败。<br />JUnit 4<br />  JUnit 框架主要聚焦于测试的隔离，暂时还不支持这个特性。<br />TestNG<br />  它使用`dependOnMethods`来实现了依赖测试的功能，如下：

```java
@Test
public void method1() {
  System.out.println("This is method 1");
}

@Test(dependsOnMethods={"method1"})
public void method2() {
	System.out.println("This is method 2");
}

```

  如果`method1()`成功执行，那么`method2()`也将被执行，否则`method2()`将会被忽略。<br />  从以上可见，大部分场景下，选择任意一个测试框架基本能满足测试场景。
<a name="ddd22119"></a>
#### 性能测试
TestNG支持通过多个线程并发调用一个测试接口来实现性能测试。JUnit4不支持，若要进行性能测试需手动添加并发代码。

```java
@Test(invocationCount=1000, threadPoolSize=5, timeOut=100)
public void perfMethod() {
    System.out.println("This is perfMethod");
}
```
<a name="b4da0f81"></a>
#### 并行测试
TestNG支持通过多个线程并发调用多个测试接口执行测试，相对于传统的单线程执行测试的方式，可以很大程度减少测试运行时间。

```java
public class ConcurrencyTest {
    @Test
    public void method1() {
        System.out.println("This is method 1");
    }
    @Test
    public void method2() {
        System.out.println("This is method 2");
    }
}
```
并行测试配置：

```xml
<suite name="Concurrency Suite" parallel="methods" thread-count="2" >
  <test name="Concurrency Test" group-by-instances="true">
    <classes>
      <class name="wow.unit.test.ConcurrencyTest" />
    </classes>
  </test>
</suite>
```

通过上面的对比，建议使用TestNG作为Java项目的单元测试框架，因为TestNG在参数化测试、依赖测试以、套件测试（组）及并发测试方面功能更加简洁、强大。另外，TestNG也涵盖了JUnit4的全部功能。但是如果业务没有这些要求那么使用junit也是可以的。

<a name="15983f67"></a>
### mock测试框架及差异
<a name="71a6b035"></a>
#### Mock的使用场景
比如Mock以下场景：
1. 外部依赖的应用的调用，比如WebService等服务依赖。
1. DAO层（访问MySQL、Oracle、Emcache等底层存储）的调用等。
1. 系统间异步交互通知消息。
1. methodA里面调用到的methodB。
1. 一些应用里面自己的Class(abstract，final，static）、Interface、Annotation、Enum和Native等。
<a name="45f4270c"></a>
#### Mock工具的原理
![image.png]({{ site.url }}/assets/qte/mock原理.png)
1. Record阶段：录制期望。也可以理解为数据准备阶段。创建依赖的Class或Interface或Method，模拟返回的数据、耗时及调用的次数等。
1. Replay阶段：通过调用被测代码，执行测试。期间会Invoke到第一阶段Record的Mock对象或方法。
1. Verify阶段：验证。可以验证调用返回是否正确，及Mock的方法调用次数，顺序等。

<a name="b13e835f"></a>
#### 主流框架对比
  我们常用的mock测试框架主要有mockito、jmockit、powermock、easymock、jmock，其中主要功能对比下：<br />![image.png]({{ site.url }}/assets/qte/主流mock框架对比.png)<br />

从上面可以看到，JMockit的的功能最全面、强大！它基于java.lang.instrument包开发，并使用ASM库来修改Java的Bytecode，因此基本能实现无所不能的mock。然而由于它的学习成本相对较高，因此目前主流的mock测试框架为jmockit和mockito，下面来具体看看他们的优劣点：

| 序号 | mockito | jmockit |
| --- | --- | --- |
| 1 | 本身不支持静态方法或构造函数的mock，但结合powermock可以实现 | 能直接mock静态方法或构造函数 |
| 2 | mock API使用前后不太一直，具体表现为：在record阶段，我们调用类似这样调用when(mock.mockedMethod(args))；然而在verify阶段，我们却这样调用verify(mock).mockedMethod(args)<br />前者我们调用被mock方法是直接调用mock对象，而后者是调用verify返回的对象 | 没有这种不一致，调用被mock方法永远是调用mock对象本身 |
| 3 | 没有内建的代码覆盖率工具，使用结合jacoco和mockito+powermock做代码覆盖率报告是冲突和问题较多。而Cobertura的支持更好 | 包含内建的代码覆盖率工具，并且完美支持jacoco |
| 4 | 相对jmockit更容易学习和使用 | 学习曲线比较陡 |
| 5 | 自定义参数匹配相比jmockit更加复杂 | 相对简单 |
| 6 | 名气更大，更受欢迎，社区更大，作为spring的官方mock框架 | 知名度和社区相对小一些 |
| 7 | 使用代理API的设计架构 | 基于Java 1.5 instrumentation API  |
| 8 | 开源<br />[https://github.com/mockito](https://github.com/mockito) | 开源<br />[http://jmockit.github.io/](http://jmockit.github.io/) |

**因此如果没有很特别的需求，使用mockito更好！！**

---- 扩展<br />JMockit有两种Mock的方式：
* Behavior-oriented(Expectations & Verifications)
* State-oriented(MockUp)

通俗点讲，Behavior-oriented是基于行为的Mock，对Mock目标代码的行为进行模仿，像是黑盒测试。State-oriented是基于状态的Mock，是站在目标测试代码内部的。可以对传入的参数进行检查、匹配，才返回某些结果，类似白盒。而State-oriented的new MockUp基本上可以Mock任何代码或逻辑。<br />----

<a name="bc90fe3f"></a>
## 四、代码覆盖率
  经常有人问这样的问题：“我们在做单元测试，那测试覆盖率要到多少才行？”。答案其实很简单，“作为指标的测试覆盖率都是没有用处的。”<br />  Martin Fowler（重构那本书的作者）曾经写过一篇博客来讨论这个问题，他指出：把测试覆盖作为质量目标没有任何意义，而我们应该把它作为一种发现未被测试覆盖的代码的手段。
<a name="28d00e07"></a>
#### 代码覆盖率的意义
1. 分析未覆盖部分的代码，从而反推在前期测试设计是否充分，没有覆盖到的代码是否是测试设计的盲点，为什么没有考虑到？需求/设计不够清晰，测试设计的理解有误，工程方法应用后的造成的策略性放弃等等，之后进行补充测试用例设计。
1. 检测出程序中的废代码，可以逆向反推在代码设计中思维混乱点，提醒设计/开发人员理清代码逻辑关系，提升代码质量。
1. 代码覆盖率高不能说明代码质量高，但是反过来看，代码覆盖率低，代码质量不会高到哪里去，可以作为测试自我审视的重要工具之一。

<a name="81cf009a"></a>
#### 代码覆盖率工具
_目前Java常用覆盖率工具Jacoco、Emma和Cobertura_

|  | Jacoco | Emma | Cobertura |
| --- | --- | --- | --- |
| 原理 | 使用asm修改字节码 | 可以修改jar文件、class文件字节码文件 | 基于Jcoverage<br />基于asm框架对class插桩 |
| 覆盖粒度 | 方法、类、行、分支、指令 | 行、方法、类 | 行、分支 |
| 插桩 | on-the-fly和offline | on-the-fly和offline | offline |
| 缺点 |  | 不支持jdk8 | 关闭服务器才能获取覆盖率报告 |
| 性能 | 快 | 较快 | 较快 |

<a name="79a10235"></a>
#### 覆盖率工具工作流程

![代码覆盖率工具工作流程.png](http://assets.processon.com/chart_image/5c8912b7e4b0afc744109c36.png)

1. 对Java字节码进行插桩，On-The-Fly和Offine两种方式。
1. 执行测试用例，收集程序执行轨迹信息，将其dump到内存。
1. 数据处理器结合程序执行轨迹信息和代码结构信息分析生成代码覆盖率报告。
1. 将代码覆盖率报告图形化展示出来，如html、xml等文件格式。
<a name="4dd57411"></a>
#### 插桩原理
![]({{ site.url }}/assets/qte/插桩原理.jpeg)

  主流代码覆盖率工具都采用字节码插桩模式，通过钩子的方式来记录代码执行轨迹信息。其中字节码插桩又分为两种模式On-The-Fly和Offine。On-The-Fly模式优点在于无需修改源代码，可以在系统不停机的情况下，实时收集代码覆盖率信息。Offine模式优点在于系统启动不需要额外开启代理，但是只能在系统停机的情况下才能获取代码覆盖率。 基于以上特性，同时由于使用JDK8，我们采用Jacoco来获取集成测试代码覆盖率，单元测试使Cobertura。

**On-The-Fly插桩 Java Agent**
* JVM中通过-javaagent参数指定特定的jar文件启动Instrumentation的代理程序
* 代理程序在每装载一个class文件前判断是否已经转换修改了该文件，如果没有则需要将探针插入class文件中。
* 代码覆盖率就可以在JVM执行代码的时候实时获取。
* 典型代表：Jacoco

**On-The-Fly插桩 Class Loader**
* 自定义classloader实现自己的类装载策略，在类加载之前将探针插入class文件中
* 典型代表：Emma

**Offine插桩**
* 在测试之前先对文件进行插桩，生成插过桩的class文件或者jar包，执行插过桩的class文件或者jar包之后，会生成覆盖率信息到文件，最后统一对覆盖率信息进行处理，并生成报告。
* Offline插桩又分为两种：
  * Replace：修改字节码生成新的class文件
  * Inject：在原有字节码文件上进行修改
* 典型代表：Cobertura

**On-The-Fly和Offine比较**
* On-The-Fly模式更加方便的获取代码覆盖率，无需提前进行字节码插桩，可以实时获取代码覆盖率信息
* Offline模式适用于以下场景：
  * 运行环境不支持java agent
  * 部署环境不允许设置JVM参数
  * 字节码需要被转换成其他虚拟机字节码，如Android Dalvik VM
  * 动态修改字节码过程中和其他agent冲突
  * 无法自定义用户加载类

<a name="e6bec5dc"></a>
## 五、持续集成
  说到持续集成，相信大家脑海里就会想起Jenkins，典型的持续集成流程如下：<br />
![持续集成流程.png](http://assets.processon.com/chart_image/5c8a4387e4b09a16b9a2bae2.png)

互联网软件的开发和发布，已经形成了一套标准流程，最重要的组成部分就是持续集成（Continuous integration，简称CI）。

  持续集成的目的，就是让产品可以快速迭代，同时还能保持高质量。它的核心措施是，代码集成到主干之前，必须通过自动化测试。只要有一个测试用例失败，就不能集成。<br />  Martin Fowler说过，"持续集成并不能消除Bug，而是让它们非常容易发现和改正。"<br />与持续集成相关的，还有两个概念，分别是持续交付和持续部署。

<a name="d3c5ef07"></a>
#### 持续交付
  持续交付（Continuous delivery）指的是，频繁地将软件的新版本，交付给质量团队或者用户，以供评审。如果评审通过，代码就进入生产阶段。<br />  持续交付可以看作持续集成的下一步。它强调的是，不管怎么更新，软件是随时随地可以交付的。
<a name="cd738654"></a>
#### 持续部署
  持续部署（continuous deployment）是持续交付的下一步，指的是代码通过评审以后，自动部署到生产环境。<br />  持续部署的目标是，代码在任何时刻都是可部署的，可以进入生产阶段。<br />  持续部署的前提是能自动化完成测试、构建、部署等步骤。它与持续交付的区别，可以参考下图。

![image.png]({{ site.url }}/assets/qte/持续交付.png)

常用的构建工具如下。
> * [Jenkins](http://jenkins-ci.org/)
* [Travis](https://travis-ci.com/)
* [Codeship](https://www.codeship.io/)
* [Strider](http://stridercd.com/)

  Jenkins和Strider是开源软件，Travis和Codeship对于开源项目可以免费使用。它们都会将构建和测试，在一次运行中执行完成。

<a name="fd5c6fbe"></a>
## 六、扩展
1、代码安全扫描<br />2、变异测试<br />3、增量代码sonar<br />...
