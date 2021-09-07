---
highlight: darcula
theme: smartblue
---

# koa-compose-analysis

## 1. 前言

大家好，我是[若川](https://lxchuan12.gitee.io)。欢迎关注我的[公众号若川视野](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/12/13/16efe57ddc7c9eb3~tplv-t2oaga2asx-image.image "https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/12/13/16efe57ddc7c9eb3~tplv-t2oaga2asx-image.image")，最近组织了[**源码共读活动**《1个月，200+人，一起读了4周源码》](https://mp.weixin.qq.com/s?__biz=MzA5MjQwMzQyNw==&mid=2650756550&idx=1&sn=9acc5e30325963e455f53ec2f64c1fdd&chksm=8866564abf11df5c41307dba3eb84e8e14de900e1b3500aaebe802aff05b0ba2c24e4690516b&token=917686367&lang=zh_CN#rd)，感兴趣的可以加我微信 [ruochuan12](https://mp.weixin.qq.com/s?__biz=MzA5MjQwMzQyNw==&mid=2650756550&idx=1&sn=9acc5e30325963e455f53ec2f64c1fdd&chksm=8866564abf11df5c41307dba3eb84e8e14de900e1b3500aaebe802aff05b0ba2c24e4690516b&token=917686367&lang=zh_CN#rd) 参与，长期交流学习。

之前写的[《学习源码整体架构系列》](https://juejin.cn/column/6960551178908205093) 包含`jQuery`、`underscore`、`lodash`、`vuex`、`sentry`、`axios`、`redux`、`koa`、`vue-devtools`、`vuex4`十篇源码文章。

写相对很难的源码，耗费了自己的时间和精力，也没收获多少阅读点赞，其实是一件挺受打击的事情。从阅读量和读者受益方面来看，不能促进作者持续输出文章。

所以转变思路，写一些相对通俗易懂的文章。**其实源码也不是想象的那么难，至少有很多看得懂**。

之前写过 koa 源码文章比较长，大概率看不完。[学习 koa 源码的整体架构，浅析koa洋葱模型原理和co原理](https://juejin.cn/post/6844904088220467213)

本文涉及到的 [koa-compose 仓库](https://github.com/koajs/compose) 文件，整个`index.js`文件代码行数虽然不到 `50` 行，而且测试用例`test/test.js`文件 `300` 余行，但非常值得我们学习。

歌德曾说：读一本好书，就是在和高尚的人谈话。 同理可得：读源码，也算是和作者的一种学习交流的方式。

阅读本文，你将学到：

```bash
1. 熟悉 koa-compose 中间件源码、可以应对面试官相关问题
2. 学会使用测试用例调试源码
3. 学会 jest 部分用法
```

## 2. 环境准备

### 2.1 克隆 koa-compose 项目

```bash
# 可以直接克隆我的仓库，我的仓库保留的 compose 仓库的 git 记录
git clone https://github.com/lxchuan12/koa-compose-analysis.git
cd koa-compose/compose
npm i
```

顺带说下：我是怎么保留 `compose` 仓库的 `git` 记录的。

```bash
# 在 github 上新建一个仓库 `koa-compose-analysis` 克隆下来
git clone https://github.com/lxchuan12/koa-compose-analysis.git
cd koa-compose-analysis
git subtree add --prefix=compose https://github.com/koajs/compose.git main
# 这样就把 compose 文件夹克隆到自己的 git 仓库了。且保留的 git 记录
```

关于更多 `git subtree`，可以看这篇文章[用 Git Subtree 在多个 Git 项目间双向同步子项目，附简明使用手册](https://segmentfault.com/a/1190000003969060)

接着我们来看怎么根据开源项目中提供的测试用例调试源码。

### 2.2 根据测试用例调试 compose 源码

用`VSCode`（我的版本是 `1.60` ）打开项目，找到 `compose/package.json`，找到 `scripts` 和 `test` 命令。

```json
// compose/package.json
{
    "name": "koa-compose",
    // debug （调试）
    "scripts": {
        "eslint": "standard --fix .",
        "test": "jest"
    },
}
```

在`scripts`上方应该会有`debug`或者`调试`字样。点击`debug`(调试)，选择 `test`。

![VSCode 调试](./images/scripts-test-debugger.png)

接着会执行测试用例`test/test.js`文件。终端输出如下图所示。

![koa-compose 测试用例输出结果](./images/jest-ternimal.png)

接着我们调试 `compose/test/test.js` 文件。
我们可以在 `45行` 打上断点，重新点击 `package.json` => `srcipts` => `test` 进入调试模式。
如下图所示。

![koa-compose 调试](./images/test-compose-debugger.png)

接着按上方的按钮，继续调试。在`compose/index.js`文件中关键的地方打上断点，调试学习源码事半功倍。

[更多 nodejs 调试相关 可以查看官方文档](https://code.visualstudio.com/docs/nodejs/nodejs-debugging)

TODO: 顺便提一下按钮。

## 3. 跟着测试用例学源码

提一个测试用例小技巧：我们可以在第一个测试用例处加上`only`修饰。

```js
it.only('should work', async () => {})
```

这样我们就可以只执行当前的测试用例，不关心其他的，不会干扰调试。

### 3.1 正常流程

```js
// compose/test/test.js
'use strict'

/* eslint-env jest */

const compose = require('..')
const assert = require('assert')

function wait (ms) {
  return new Promise((resolve) => setTimeout(resolve, ms || 1))
}
// 分组
describe('Koa Compose', function () {
  it.only('should work', async () => {
    const arr = []
    const stack = []

    stack.push(async (context, next) => {
      arr.push(1)
      await wait(1)
      await next()
      await wait(1)
      arr.push(6)
    })

    stack.push(async (context, next) => {
      arr.push(2)
      await wait(1)
      await next()
      await wait(1)
      arr.push(5)
    })

    stack.push(async (context, next) => {
      arr.push(3)
      await wait(1)
      await next()
      await wait(1)
      arr.push(4)
    })

    await compose(stack)({})
    // 最后输出数组是 [1,2,3,4,5,6]
    expect(arr).toEqual(expect.arrayContaining([1, 2, 3, 4, 5, 6]))
  })
}
```

大概看完这段测试用例，`context`是什么，`next`又是什么。

在[`koa`的文档](https://github.com/koajs/koa/blob/master/docs/guide.md#writing-middleware)上有个非常代表性的中间件 `gif` 图。

![中间件 gif 图](./images/middleware.gif)

而`compose`函数作用就是把添加进中间件数组的函数按照上面 `gif` 图的顺序执行。

#### 3.1.1 compose 函数

简单来说，`compose` 函数主要做了两件事情。

- 1. 接收一个参数，校验参数是数组，且校验数组中的每一项是函数。
- 2. 返回一个函数，这个函数接收两个参数，分别是`context`和`next`，这个函数最后返回`Promise`。

```js
/**
 * Compose `middleware` returning
 * a fully valid middleware comprised
 * of all those which are passed.
 *
 * @param {Array} middleware
 * @return {Function}
 * @api public
 */
function compose (middleware) {
  // 校验传入的参数是数组，校验数组中每一项是函数
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch(i){
      // 省略，下文讲述
    }
  }
}
```

接着我们来看 `dispatch` 函数。

#### 3.1.2 dispatch 函数

```js
function dispatch (i) {
  if (i <= index) return Promise.reject(new Error('next() called multiple times'))
  index = i
  let fn = middleware[i]
  if (i === middleware.length) fn = next
  if (!fn) return Promise.resolve()
  try {
    return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))
  } catch (err) {
    return Promise.reject(err)
  }
}
```

#### 3.1.3 简化 compose 便于理解

省略

```js
// 这样就可能更好理解了。
// simpleKoaCompose
const [fn1, fn2, fn3] = stack;
const fnMiddleware = function(context){
    return Promise.resolve(
      fn1(context, function next(){
        return Promise.resolve(
          fn2(context, function next(){
              return Promise.resolve(
                  fn3(context, function next(){
                    return Promise.resolve();
                  })
              )
          })
        )
    })
  );
};
```

>也就是说`koa-compose`返回的是一个`Promise`，`Promise`中取出第一个函数，传入`context`和第一个`next`函数来执行。<br>
第一个`next`函数里也是返回的是一个`Promise`，`Promise`中取出第二个函数，传入`context`和第二个`next`函数来执行。<br>
第二个`next`函数里也是返回的是一个`Promise`，`Promise`中取出第三个函数，传入`context`和第三个`next`函数来执行。<br>
第三个...<br>
以此类推。最后一个中间件中有调用`next`函数，则返回`Promise.resolve`。如果没有，则不执行`next`函数。
这样就把所有中间件串联起来了。这也就是我们常说的洋葱模型。<br>

**不得不说非常惊艳，“玩还是大神会玩”**。

### 3.2 错误捕获

```js
it('should catch downstream errors', async () => {
  const arr = []
  const stack = []

  stack.push(async (ctx, next) => {
    arr.push(1)
    try {
      arr.push(6)
      await next()
      arr.push(7)
    } catch (err) {
      arr.push(2)
    }
    arr.push(3)
  })

  stack.push(async (ctx, next) => {
    arr.push(4)
    throw new Error()
  })

  await compose(stack)({})
  // 输出顺序 是 [ 1, 6, 4, 2, 3 ]
  expect(arr).toEqual([1, 6, 4, 2, 3])
})
```

### 3.3 next 函数不能调用多次

```js
it('should throw if next() is called multiple times', () => {
  return compose([
    async (ctx, next) => {
      await next()
      await next()
    }
  ])({}).then(() => {
    throw new Error('boom')
  }, (err) => {
    assert(/multiple times/.test(err.message))
  })
})
```

其他还有很多测试用例可以按照文中方法自行调试，相信学会了调试后觉得源码也没有想象中的那么难。

## 4. 总结

开源项目，一般都会有很全面的测试用例。
