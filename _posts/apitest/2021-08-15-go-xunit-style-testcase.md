---
layout: post
title: Go实现xunit风格的用例编写
categories: [自动化测试]
description: 使用go实现xunit风格的用例编写
keywords: go, xunit, rpc
published: true
---



# Grpc接口?

由于公司的技术栈选择，大多数业务团队都是使用go作为后端开发语言，我们也不例外，同时我们是偏基础的业务，因此大部分是grpc接口，对外的http接口较少。随着团队的不断扩张，测试回归的需求越来越大，实现自动化回归迫在眉睫。因为这些背景，使用go作为实现接口自动化测试的语言就是顺其自然的事情了。

# go testing?

Go语言自带测试模块: [testing](https://golang.org/pkg/testing/) 。执行器、断言使得该模块可以很好的满足日常的单元测试，同时也有很多第三方模块对断言进行了增强，以及增加了一些mock功能，甚至实现流行的BDD模式。这些框架有：Testify、GoConvey、Ginkgo等。对go的各种测试框架了解下来后发现，在go下实现接口测试过于自由，代码的复用性不高，比如将有前后条件的的一些用例组织在一起是一个问题。

# 如何处理前后置条件？

举个简单的例子，假如你在编写一个成功获取直播播放的地址用例，根据用例设计的原子性和独立性原则，需要在执行此用例前先开播，然后在执行后把直播关闭了，很多用例可能都需要基于开播后才能执行，最后所有用例执行完成 后，关闭这场直播。

当然，可以单独实现开播和结束直播的方法，然后在每个测试用例的开始和结尾调用，不过这非常不合理。写过JAVA的人应该知道junit或者testng，这些测试框架都提供了@Before、@After、@BeforeClass、@AfterClass的注解，可以在每个测试方法执行前后或，所有测试方法执行前后添加一些操作。因此打算在go中找到这种类xunit类型的框架。

# Gunit

根据上面的想法，找到了go实现的xunit框架：[gunit](https://github.com/smartystreets/gunit)。

Gunit实现了每个测试方法前后执行操作，但是并未实现在所有测试方法执行前后执行操作，所以需要对其进行改造。实现类似以下 的用例编写方式：

```go
func TestMyScene(t *testing.T) {
   gunit.Run(new(MyFixture), t)
}

type MyFixture struct {
   *gunit.Fixture
}
// 所有Test方法执行前执行
func (g *MyFixture) FixtureSetup() {
   log.Println("in FixtureSetup...")
}

// 所有Test方法执行后执行
func (g *MyFixture) FixtureTeardown() {
   log.Println("in FixtureTearDown...")
}

// 每一个Test方法执行前执行
func (g *MyFixture) Setup() {
   log.Println("in test setup...")
}

// 每一个Test方法执行后执行
func (g *MyFixture) Teardown() {
   log.Println("in test teardown...")
}

// 真正的测试方法A
func (g *MyFixture) TestA() {
   g.T().Log("hello TestA...")
}

// 真正的测试方法B
func (g *MyFixture) TestB() {
   g.T().Log("hello TestB...")
}

```

# 测试报告

目前直接执行以下命令输出测试结果：

```shell
go test ./testcases/...
```



# 持续测试

当用例能够在命令行稳定执行后，接下来就是接入jenkins进行持续测试，融入到整个研测流程中。

