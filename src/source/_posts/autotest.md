---
title: 自动化测试
date: 2020-09-11 15:11:35
tags: auto test
description: Unit, Integration, UI, End-to-end
---

# 背景
质量保证在软件开发中一直占据着重要地位，如何在快速开发软件的过程中保证质量，是软件开发工程师一直面临的问题。
传统人工测试采用在测试环境进行黑盒测试的技术方案，软件系统规模越大，局限性越明显。自动化测试可以补充传统人工测试的短板，集成到构建部署流程中，自动执行并生成测试报告，减少传统人工测试中的惯性思维所导致的测试疏漏和人为差错。

# 测试模型
在自动化测试中，有一个占据重要地位的模型：测试金字塔。该概念由 Mike Cohn 于 1990 年在 [Succeeding with Agile](https://www.amazon.com/Succeeding-Agile-Software-Development-Using/dp/0321579364) 一书中首次提出。测试金字塔形象的描述了测试分层的概念及特性。
<img src="/images/auto_test_model.png" width="100%">
上图中的测试层级太少，而且很多人认为其中层级的命名和概念不够理想。可以忽略图中的测试层级，关注测试金字塔模型中的重要信息：
1. 测试层级越高，测试用例数应该越少；
2. 测试层级越高，测试用例集成度越高，独立性越差；
3. 测试层级越高，测试用例执行速度越慢；

# 测试层级

## 概念
在上面的测试模型中，引入了测试分层的概念，这里先介绍常见测试层级的概念(层级由低到高)。

### Unit Tests
单元测试是对源代码中的最小可测试单元进行检查和验证。通常而言：一个单元测试是用于判断某个特定条件（或者场景）下某个特定函数的行为是否符合预期。

### Integration Tests
集成测试层级高于单元测试，涵盖了系统性能、功能和可靠性等所有方面。通常，小型软件系统只需要单次集成测试，而大型系统需要划分为多个集成阶段，例如：将模块集成到低级子系统，然后将低级子系统集成到大型子系统。

### UI Tests
界面测试是检测应用中的用户界面是否符合预期。例如：用户的输入触发正确的动作、数据正确展示、UI 状态正确变化等。

### End-to-end Tests
端到端测试是在用户使用的真实场景中，测试软件的运行是否符合预期。例如：与数据库，网络，硬件和其他应用程序进行通信。

## 实践经验
接下里介绍在设计不同层级的自动化测试时，值得借鉴的一些实践经验。

### Unit Tests
在自动化测试中，大部分测试用例应该集中在单元测试，因为单元测试运行速度更快、独立性更强。此外，良好的单元测试设计能指导重构的方向，保证重构的质量。
那么，什么是良好的单元测试设计呢？**单元测试应该覆盖代码的所有路径（包括正常路径和边缘路径）；同时和业务代码解耦，忽略业务代码具体实现的测试，测试可观测的行为。**
示例如下：
（正确做法）如果我的输入是 x 和 y，输出会是 z 吗？
（错误做法）如果我的输入是 x 和 y，那么这个方法会先调用 A 类，然后调用 B 类，接着输出 A 类和 B 类返回值相加的结果吗？

### End-to-end Tests
端到端测试是对整个完全集成的系统进行测试，最大程度的给予开发人员信心。但端到端测试运行速度慢、独立性差。独立性差导致端到端测试通常比较脆弱，经常因为意料之外的问题导致运行结果不符合预期，且提供的错误信息一般无法直接定位到问题。此外，还增加了维护成本，被测系统的更新很可能附带着额外的更新端到端测试的工作。
根据端到端测试的上述特性，**应该尽量将端到端测试的数量减少到最低限度。考虑应用中高价值的交互和产品的核心功能，然后为其中最重要的部分编写自动化的端到端测试。**
个人观点：端到端测试应该和人工测试组合使用，而不是替代人工测试。项目开发应该主要通过单元测试和人工测试保证质量。端到端测试作为确保线上核心功能正常运行的最后一道防线，帮助开发测试同学及时发现核心功能的异常。

# Test Double（测试替身）
在写自动化测试（尤其是单元测试）时，被测试主体一般存在对其他模块或网络请求的依赖。为了减少自动化测试的副作用和复杂的测试准备，替换被测主体的依赖是常见需求。存在通用术语 Test Double 描述这种需求，可翻译为测试替身。

## 类型
测试替身有如下几种常见类型。了解不同类型的命名和的概念，在面临测试替身的需求时，有助于归纳类型和解决方案；在使用库或框架提供的测试替身对应类型的解决方案时，对类型概念也更加清晰。
<img src="/images/test_double_type.png" width="100%">

### Dummy Object（虚拟对象）
被测系统的某些方法接收对象作为参数，如果测试和被测系统都不关心这个对象，可以选择传入一个虚拟对象，该对象可能是一个简单的空对象引用或对象实例。

### Test Stub（测试存根）
测试存根被用于替换被测系统依赖的实际组件，以便控制自动化测试按照预期的路径执行。

### Test Spy（测试间谍）
测试间谍被用于捕获被测系统的输入、输出等，并将其保存起来供测试验证。

### Mock Object（模拟对象）
模拟对象被用于验证被测系统执行后的输出。通常，模拟对象包括了测试存根的功能，因为它必须返回值给被测系统。但模拟对象的重点是验证输出。模拟对象不仅是测试存根加断言的组合，使用方式也是完全不同的。

### Fake Object（伪造对象）
伪造对象被用于替换在测试环境中不存在或有副作用的对象。

## 实践
Sinon 是 Node.js 社区中较为流行的测试替身库，为 JavaScript 提供了独立的 Spy 、Stub 和 Mock，可与任何测试框架组合使用。下面通过简单介绍和使用 Sinon 中提供的不同类型解决方案，了解上述测试替身类型在实践中的应用。此外，根据 Sinon 源码提取了其中部分类型的简单实践，进一步了解测试替身类型的解决方案。

### [Sinon](http://sinonjs.org/)

#### Spy
测试间谍是一个函数，用于记录调用函数的参数、返回值、this 的值以及异常。分为两类：匿名函数和被测系统中已存在的方法的包装函数。测试间谍用于监听，不影响被测函数的正常调用。

**适用场景**
监听函数被调用的参数、返回值、this 的值以及异常。
**使用示例**
https://github.com/freezer95/autotest/tree/master/packages/sinon-mocha/test/spies.js
**简单实现**
https://github.com/freezer95/autotest/tree/master/packages/sinon-mocha/lib/spy.js

#### Stub
测试存根是带有预编程行为的函数。除了支持改变存根行为的方法，还完整支持 Spy API。类似 Spy，存根可以是匿名的，也可以封装现有的函数。当使用存根包装现有函数时，原始函数不会被调用。
**适用场景**
1. 在测试中控制方法的行为，使代码沿着特定的路径运行。如：为测试错误处理强制方法抛出错误。
2. 希望阻止特定的方法的直接调用。如：该方法会触发 XMLHttpRequest 等在测试环境下不希望触发的行为。

**使用示例**
https://github.com/freezer95/autotest/tree/master/packages/sinon-mocha/test/stubs.js
**简单实现**
https://github.com/freezer95/autotest/tree/master/packages/sinon-mocha/lib/stub.js

#### Mock
模拟是带有预编程行为和期望的伪造函数。支持 Stub API 和 Spy API。
**适用场景**
1. 先声明期望，支持链式调用，而不是在事实发生后断言

**使用示例**
https://github.com/freezer95/autotest/tree/master/packages/sinon-mocha/test/mocks.js

# 测试框架
[https://www.npmtrends.com/mocha-vs-jest-vs-jasmine-vs-ava-vs-qunit-vs-tape](https://www.npmtrends.com/mocha-vs-jest-vs-jasmine-vs-ava-vs-qunit-vs-tape)

## Mocha
Mocha 是一个能够运行在 Node.js 和浏览器中的 JavaScript 测试框架。按顺序执行测试用例，支持异步测试，支持可扩展的 reporters。
- **社区** - 提供了各种特殊场景可用的插件或扩展
- **可扩展性** - 与插件、扩展、第三方库等组合使用，可以支持各种特性

## Jest
Jest 是 Facebook 推荐使用的测试框架，它基于 Jasmine。Facebook [重写了大部分的功能](https://jestjs.io/blog/2016/09/01/jest-15.html)，还加上了很多新特性。
- **性能** - [并行测试多文件](https://jestjs.io/blog/2016/03/11/javascript-unit-testing-performance.html)，提高了运行速度
- **开箱即用** - 内置了好用的 expect 断言、mock（类似 Sinon 能提供的能力）、代码覆盖工具等，简单配置即可使用。
- **快照测试** - 提供了 [Snapshot Test](https://github.com/facebook/jest/tree/master/packages/jest-snapshot) 的能力，确保 UI 不会意外更改

# 测试工具

## [Karma](https://karma-runner.github.io/2.0/index.html)
>A simple tool that allows you to execute JavaScript code in multiple real browsers.

Karma 是一个基于 Node.js 的 JavaScript 测试执行过程管理工具（Test Runner），不是测试框架，也不是断言库。 Karma 只是启动 HTTP 服务器，并执行从其他测试框架中生成的文件。
- **真实浏览器和设备** - 简单配置即可在主流浏览器和真实设备（包括手机、平板等）中自动化执行测试代码（支持调试）。支持：Chrome、Firefox、Safari、Opera、IE、PhantomJS、ChromeHeadless、ChromeCanary。
- **开发立即更新** - 监控文件变化并自动执行，开发人员在本地修改代码后可即时获得执行结果。
- **集成喜欢的测试框架** - 可选择在支持范围内，集成喜欢的测试框架，支持 [Jasmine](https://github.com/karma-runner/karma-jasmine)、[Mocha](https://github.com/karma-runner/karma-mocha)、[QUnit](https://github.com/karma-runner/karma-qunit) and [many others](https://www.npmjs.org/browse/keyword/karma-adapter)。

### 快速上手
https://github.com/freezer95/autotest/tree/master/packages/karma

# End-to-end 测试
<img src="/images/end_to_end_framework.png" width="100%">

根据实现的底层依赖，端到端测试的解决方案可划分为两类： WebDriver、非 WebDriver。
WebDriver 类支持所有主流浏览器，以 [Nightwatch](https://nightwatchjs.org/) 为代表具体介绍。
非 WebDriver 类只能支持相同内核浏览器，以 [Puppeteer](https://developers.google.com/web/tools/puppeteer) 为代表具体介绍。

## [Nightwatch](https://nightwatchjs.org/)
> Nightwatch.js is an integrated, easy to use End-to-End testing solution for web applications and websites, written in Node.js. It uses the W3C WebDriver API to drive browsers in order to perform commands and assertions on DOM elements.

基于 WebDriver，支持主流浏览器
- **开箱即用** - 内置断言 expect 和 assert，支持 describe，提供了同步和异步的测试钩子
- **元素定位** - 支持 CSS  选择器和 [XPath](https://www.w3school.com.cn/xpath/xpath_syntax.asp) 定位元素，也可使用其他定位器策略，详见 [WebDriver Doc](https://www.w3.org/TR/webdriver/#locator-strategies)
```javascript
module.exports = {
  demoTest: function (browser) {
    browser
      .click({
        selector: '//tr[@data-recordid]/span[text()='Search Text']',
        locateStrategy: 'xpath'
      })
      .setValue('input[type=text]', 'nightwatch');
  }
};
```

### 快速上手
https://github.com/freezer95/autotest/tree/master/packages/nightwatch

## [Puppeteer](https://developers.google.com/web/tools/puppeteer)（US /ˌpʌp·ɪˈtɪər/）
>Puppeteer is a Node library which provides a high-level API to control [headless](https://developers.google.com/web/updates/2017/04/headless-chrome) Chrome or Chromium over the [DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/). It can also be configured to use full (non-headless) Chrome or Chromium.

基于 chromium，主流浏览器仅支持 chrome
- **屏幕截图** - 生成页面的屏幕截图和pdf文件
- **元素定位** - 支持 CSS 选择器定位
- **自动交互** - 提供自动执行表单提交，UI测试，键盘输入的 API，如 [page.type](https://pptr.dev/#?product=Puppeteer&version=v3.0.4&show=api-pagetypeselector-text-options)

### 快速上手
https://github.com/freezer95/autotest/tree/master/packages/puppeteer

# 参考文档
[The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)（翻译版：[测试金字塔实战](https://insights.thoughtworks.cn/practical-test-pyramid/)）
[Explore a more complete front-end testing strategy](https://medium.com/@unadlib/explore-a-more-complete-front-end-testing-strategy-e60b07cee621)
[Test Double](http://xunitpatterns.com/Test%20Double.html)
[自动化测试-e2e测试框架选择](https://juejin.im/post/5e1d8900e51d451c8771c6cd)
[前端自动化测试概览](https://juejin.im/entry/5b286a126fb9a00e45113435)
