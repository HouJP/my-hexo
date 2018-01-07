---
title: Unit Test中的Mock Objects
date: 2018-01-07 12:10:18
tags: [java, unit-testing]
---

### What's Mock Objects

Mock objects 是 test double 的另一种称呼，包含以下三种类型的对象：

1. Mocks
2. Stubs
3. Fakes

<!-- more -->

#### Mocks

Mock在单元测试中用来追踪对象的方法是否被正确调用：

1. 方法的调用次数是否符合预期
2. 传递给方法的参数是否正确

#### Stubs

当我们不需要知道对象内部的行为情况的时候使用Stub对象，它会返回我们告诉它返回的值。

#### Fakes

Fake是对API的轻量级实现，不适用于真正的产品。

### Testing State VS Testing Interactions

在单元测试中有两种测试的方式：

1. Testing state: 测试代码返回的结果的正确性
   * 返回结果正确
   * 系统中状态值正确改变
2. Testing interactions: 代码调用了希望被调用的某些方法
   * 方法是否被调用
   * 传递给被调用方法的参数是否正确

总结：

* Testing interactions只能保证函数被正确调用，Testing state才能真正保证结果的正确性。
* Testing interactions不利于测试代码的维护。

### 测试三部曲

单元测试分为三个阶段：

1. Arrange: 测试条件准备阶段，也就是"Given"。
2. Act: 执行阶段，调用测试方法，也就是"When"。
3. Assert: 判断结果，也就是"Then"。