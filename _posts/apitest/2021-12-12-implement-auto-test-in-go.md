---
layout: post
title: Go语言在直播业务场景的自动化测试实践
categories: [自动化测试]
description: 使用go实现自动化测试
keywords: go, xunit, autotest
published: true
---

在上一篇文章中我提到了使用xunit风格的方式编写测试用例，实际上是用go语言实现自动化测试，这段时间有了更多的实践，因此在这篇文章中对整个实现做一下梳理。

# 一、背景
目前业务的规划方向基本确定，包含上行收流、直播调度控制器、流下行三个主要方向（直播基础团队），服务接口趋于稳定，适合实现接口自动化，减少回归验证工作，并保障业务质量。

# 二、目标
实现直播核心流程接口自动化测试，接入研测流程，在日常流程中可快速验证核心功能，可作为提测和发布流程中的质量卡点。

# 三、实现内容
### 测试框架选择与改造
#### 执行框架
xunit风格的go测试框架gunit，使用结构体组合的方式，达到类似Python或JAVA中的类继承的作用，从而复用代码。然而只支持单个方法的setup、teardown，在直播场景下，需要全局的setup和teardown，类似JAVA中的@BeforeClass，比如测试拉流相关的接口，需要先推流，验证结束后，关闭推流。
因此，需要改造gunit框架，添加FixetureSetup、FixtureTeardown方法，代码结构类似如下：
```
package testcases

import (
	"github.com/smartystreets/gunit"
    "git.garena.com/shopee/live-streaming/livetech_qa/livetech_api_autotest/common/livetech"
    ...
)

func TestMyScene(t *testing.T) {
	gunit.Run(new(MyFixture), t, gunit.Options.SequentialTestCases())
}

type MyFixture struct {
	livetech.MMCHelper
}

// 所有Test方法执行前执行
func (g *MyFixture) FixtureSetup() {
	g.Info().Msg("in FixtureSetup...")
}

// 所有Test方法执行后执行
func (g *MyFixture) FixtureTeardown() {
	g.Info().Msg("in FixtureTearDown...")
}

// 每一个Test方法执行前执行
func (g *MyFixture) Setup() {
	g.Info().Msg("in test setup...")
}

// 每一个Test方法执行后执行
func (g *MyFixture) Teardown() {
	g.Info().Msg("in test teardown...")
}

// 真正的测试方法A
func (g *MyFixture) TestA() {
	g.Description("这是TestA")
	g.Info().Msg("hello TestA...")
}

// 真正的测试方法B
func (g *MyFixture) TestB() {
	g.Description("这是TestB")
	g.Info().Msg("hello TestB...")
}
```
	
#### 断言
gunit框架提供断言能力，可以覆盖大部分的断言需求，如果无法满足，扩展也很方便。

### 基础结构体
上面提到使用go结构体的组合方式复用基础代码，实现两个基础结构体，分别是MMCHelper、SLiveHelper，封装了直播调度控制器和slive(流服务)的基础操作。


### 日志打印
go test用例执行默认是并发方式的，使用-json参数可输出json格式的日志，经过实践，日志打印存在错乱的情况，比如TestA方法中打印的日志，可能在输出json后，显示是在TestB中输出的，这就给用例维护和问题排查带来很大麻烦。因此需要统一日志输出，并保证测试方法中的输出显示在报告中的正确位置。

约定输出日志为json字符串，日志打印时绑定当前测试方法名称：
​​
```
func (m *base) Log() *zerolog.Event {
  return log.Log().Str("Test", m.Name())
}

func (m *base) Debug() *zerolog.Event {
  return log.Debug().Str("Test", m.Name())
}

func (m *base) Info() *zerolog.Event {
  return log.Info().Str("Test", m.Name())
}
func (m *base) Warn() *zerolog.Event {
  return log.Warn().Str("Test", m.Name())
}

func (m *base) Error() *zerolog.Event {
  return log.Error().Str("Test", m.Name())
}
```

使用参考如下：
```
g.Error().Msgf("scan transcode tasks error:%s", err.Error())
g.Info().Msg("api test is easy")
m.Debug().Str("name", "yu.he").Msg("livetech auto test")
m.Warn().interface("task", task).Msg("livetech auto test")
```

### 流工具封装
因为要实现直播自动化测试，流操作是必不可少的，所以需要封装ffmpeg来实现推拉流，以及封装ffprobe分析流信息。

实现两个controller
```
type StreamController struct {
  FFmpeg     string
  input      string
  inputOpts  []string
  output     string
  outputOpts []string
  child      *os.Process
  Running    bool
  t          gunit.TestingT
}

type FFprobeController struct {
  FFprobe string
  t       gunit.TestingT
}
```

### 部署&执行
#### 本地执行（调试）
使用go test执行测试用例，有两种执行模式，分别是本地目录模式（local directory mode）、包列表模式（package list mode）,具体区别可以使用go help test查看。

#### 远程执行
申请新应用autoapi(作为一个用例执行容器)，用Python封装为web服务，提供http接口触发用例执行(调用子进程go test)，http接口描述如下：

```
POST https://autoapi.livetech.test.io/case_run/run_go_test  
Body:
    {"cid":"tw","concurrent_num":2,"server_name":"gatewayapi","api_name":"GetPushURLList","operator":"yu.he"}  
    
参数说明：
cid--地区 必填
concurrent_num--go test并发数，非必填
server_name--服务名，非必填，不填表示全部
api_name--接口名，非必填
operator-- 触发人，必填
```

远程执行采用包列表模式，类似：
```
go test ./testcases/... -v
```

所有用例在testcases目录下，按服务+接口组织，因此指定server_name、api_name可以只跑对应目录下的测试文件。

### 测试报告
用例执行结果按json格式输出，使用go-test-report工具进行解析，展示效果如下:

![](http://bugvanisher.cn/images/static/1639300977844.jpg)

点击 ***用例名称*** 可展开日志
![](http://bugvanisher.cn/images/static/1639316271603.jpg)

#### 持久化存储HTML
目前没有一个静态文件服务器可以保存html格式的报告，计划将文件上传到某个测试机房的虚拟机，长期保存文件(有条件的话还是保存到静态文件服务器上)。

#### 修正用例总数显示
gunit框架导致了测试报告会引入外层Test用例，总用例数量会增加test文件数，因此需要对go-test-report报告进行改造。区分ParentTest和SubTest的用例数量。

#### 持久化存储用例执行情况
得到实际的用例执行情况数量后，（总用例数、成功数量、失败数量、跳过数量），上报QA-Toolkit接口存储到数据库中，用例执行情况可追溯。（区分service、apiname）

#### 用例报告列表页
在QA-ToolKit添加两个测试报告列表页，一个是streamapi（直播调度控制器），一个是slive。
每个页面由三部分组成，分别是两个tab和一个创建任务按钮及弹框。
其中一个tab为历史执行记录图表（用例执行时间、成功率、用例总数）；另一个tab为历史执行记录列表，支持筛选。

#### 报告通知
建立通知群，创建通知机器人，开始执行和结束时发送通知到群里。结果消息包含执行时间，执行人，html报告地址。

### 持续测试
#### 自动部署
由于后台是微服务架构，直播服务核心功能覆盖多个应用，因此在执行自动化测试时需要保证核心应用部署的是目标分支的代码，改造目前的部署系统实现自动部署，并且支持部署成功回调。

#### 定时触发
选定一个固定环境，每天定时触发执行自动化测试。定时触发执行的用例为全量用例，分别是streamapi和slive。供于用例报告列表页的图表数据（全量执行用例报告）

#### 手动执行
用例报告列表页实现一个手动触发按钮，可手动输入执行参数。

整体时序图如下：
![](http://bugvanisher.cn/images/static/1639317327627.jpg)
