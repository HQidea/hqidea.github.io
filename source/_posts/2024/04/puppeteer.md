---
title: Puppeteer
date: 2024-04-10 09:09:24
tags:
---
Puppeteer是一个用Node.js编写的基于[Chrome Devtools Protocol](https://chromedevtools.github.io/devtools-protocol/)的自动化工具。它的作用有：
+ 网页截图或者生成PDF
+ 抓取单页应用（SPA）并生成预渲染内容（即SSR，服务器端渲染）
+ UI自动化测试，自动提交表单、模拟键盘输入等
+ 创建一个测试Javascript和Chrome最新特性的环境
+ 录制网站加载时间线，帮助诊断性能问题
+ 测试Chrome扩展

## 安装

```
npm i puppeteer
# or using yarn
yarn add puppeteer
# or using pnpm
pnpm i puppeteer
```

第一次安装puppeteer的时候，它会自动下载一个测试专用Chrome和一个`chrome-headless-shell`二进制文件（从Puppeteer v21.6.0版本开始提供），Chrome的下载位置可以在[配置](https://pptr.dev/api/puppeteer.configuration)中进行修改。

## API层级

![](image-4.png)

+ Puppeteer使用Chrome Devtools Protocol和浏览器通信
+ 一个浏览器实例中可以包含多个浏览器上下文，比如隐身窗口是一个独立的上下文
+ 一个浏览器上下文可以包含多个页面
+ 一个页面至少包含一个主frame，也可能包含iframe
+ 一个frame至少包含一个执行js的上下文，如果有插件的话会有多个上下文

## puppeteer vs puppeteer-core

+ puppeteer：包含`puppeteer-core`、一个测试专用Chrome以及易用的默认配置
+ puppeteer-core：通过Chrome Devtools Protocol控制浏览器的行为，不包含测试专用Chrome

+ 使用选择：

  如果你需要控制远程的浏览器，或者需要自主控制浏览器的行为，如启动一个浏览器，使用`puppeteer-core`，其他情况使用`puppeteer`即可。

## 头

Chrome中的头指的是用户看得见的界面，在2017年时Chrome 59版本中引入了无头的概念，可以在无人值守的环境中运行没有界面的Chrome。当时的实现方式是重新实现了一个浏览器，与Chrome无关，只是两者的代码和二进制包放在了一起。同时维护2个浏览器会带来很多头疼的问题，比如无头浏览器中存在Chrome中没有的bug，要想给Chrome增加新功能就得同时给无头浏览器重新实现一遍。因此，在2021年Chrome开发团队着手解决这个问题，将两者的代码进行了合并统一，在Chrome 112版本中正式发布。

![](image.png)

于是，Chrome将`--headless`参数做了升级：
```
chrome --headless=new  # 启动新的无头模式

chrome --headless      # 启动旧的无头模式，将来可能会指向新的无头模式
chrome --headless=old  # 启动旧的无头模式
```

为了兼顾无法升级的用户，Chrome将旧的无头浏览器作为单独的二进制包进行分发，也就是前文提到的`chrome-headless-shell`。

在Puppeteer中，控制浏览器是否有头的参数如下：
```
// 启动无头浏览器，版本v22之前默认启动旧的无头浏览器
const browser = await puppeteer.launch();
const browser = await puppeteer.launch({headless: true});

// 版本v22之前指定启动新的无头浏览器
const browser = await puppeteer.launch({headless: 'new'});

// 版本v22之后指定启动旧的无头浏览器
const browser = await puppeteer.launch({headless: 'shell'});

// 启动有头浏览器
const browser = await puppeteer.launch({headless: false});
```

旧的无头浏览器虽然存在一些问题，但是由于不依赖X11/Wayland, D-Bus等框架，它的性能会比Chrome更出色，因此在一些自动截屏、网页抓取等场景下仍具有一定的优势。

## 使用示例

+ 在`https://developer.chrome.com/`中搜索"automate beyond recorder"，点击第一个结果，然后在终端中打印标题。

  ```
  import puppeteer from 'puppeteer';

  (async () => {
    // Launch the browser and open a new blank page
    const browser = await puppeteer.launch();
    const page = await browser.newPage();

    // Navigate the page to a URL
    await page.goto('https://developer.chrome.com/');

    // Set screen size
    await page.setViewport({width: 1080, height: 1024});

    // Type into search box
    await page.type('.devsite-search-field', 'automate beyond recorder');

    // Wait and click on first result
    const searchResultSelector = '.devsite-result-item-link';
    await page.waitForSelector(searchResultSelector);
    await page.click(searchResultSelector);

    // Locate the full title with a unique string
    const textSelector = await page.waitForSelector(
      'text/Customize and automate'
    );
    const fullTitle = await textSelector?.evaluate(el => el.textContent);

    // Print the full title
    console.log('The title of this blog post is "%s".', fullTitle);

    await browser.close();
  })();
  ```

  结果：
  ```
  The title of this blog post is "Customize and automate user flows beyond Chrome DevTools ...".
  ```
+ 访问`https://blog.hqhome.net/`，将网页保存为pdf

  ```
  import puppeteer from 'puppeteer';

  (async () => {
    // Launch the browser and open a new blank page
    const browser = await puppeteer.launch({headless: false});
    const page = await browser.newPage();

    // Navigate the page to a URL
    await page.goto('https://blog.hqhome.net/');

    // Set screen size
    await page.setViewport({width: 1080, height: 1024});

    await page.pdf({path: 'blog.pdf', printBackground: true});

    await browser.close();
  })();
  ```

  结果：
  ![](image-1.png)

## 选择器

### P选择器

Puppeteer中的选择器被称为P选择器，它使用CSS选择器语法的超集进行元素选择，额外具有深度组合器和P元素选择器。

#### 深度组合器

+ `>>>` 会进入节点下每一个影子宿主中查找元素
+ `>>>>` 如果当前节点是一个影子宿主，则会进入其中查找元素，不然不执行操作

```
<custom-element>
  <template shadowrootmode="open">
    <slot></slot>
  </template>
  <custom-element>
    <template shadowrootmode="open">
      <slot></slot>
    </template>
    <custom-element>
      <template shadowrootmode="open">
        <slot></slot>
      </template>
      <h2>Light content</h2>
    </custom-element>
  </custom-element>
</custom-element>
```

如在上面的DOM结构中，`custom-element >>> h2`将返回`h2`，而`custom-element >>>> h2`不会返回内容，因为`h2`在更深层次的影子宿主中。

#### P元素选择器

+ 文本选择器(-p-text)：用于选择与文字匹配的第一个最深层次节点
  示例：
  ```
  const element = await page.waitForSelector('div ::-p-text(My name is Jun)');
  ```
  我实践下来，这个选择器的作用范围存在一定局限性，如下面的例子：
  ```
  const element = await page.waitForSelector('div ::-p-text(aaa)');
  ```
  ![](image-3.png)
  ![](image-2.png)
  bb中的aaa层次明显要比第一个aaa中深，但是该选择器只能在第一个命中的节点中寻找最深层次的节点
+ XPath选择器(-p-xpath)：使用XPath表达式选择元素
  示例：
  ```
  const element = await page.waitForSelector('::-p-xpath(h2)');
  ```
+ ARIA选择器(-p-aria)：使用ARIA标签选择元素
  示例：
  ```
  const node = await page.waitForSelector('::-p-aria(Submit)');
  ```
+ 自定义选择器：用户可以通过`registerCustomQueryHandler`函数实现自己的选择器逻辑
  示例：
  ```
  Puppeteer.registerCustomQueryHandler('getById', {
    queryOne: (elementOrDocument, selector) => {
      return elementOrDocument.querySelector(`[id="${CSS.escape(selector)}"]`);
    },
    // Note: for demonstation perpose only `id` should be page unique
    queryAll: (elementOrDocument, selector) => {
      return elementOrDocument.querySelectorAll(`[id="${CSS.escape(selector)}"]`);
    },
  });

  const node = await page.waitForSelector('::-p-getById(elementId)');
  // OR used in conjunction with other selectors
  const moreSpecificNode = await page.waitForSelector(
    '.side-bar ::-p-getById(elementId)'
  );
  ```

## Locators

一个会等待并自动重试失败动作的新实验性api。

+ 等待元素可见性变化（visible or hidden）
  ```
  await page.locator('button').wait();
  ```
+ 等待函数执行
  ```
  await page
    .locator(() => {
      let resolve!: (node: HTMLCanvasElement) => void;
      const promise = new Promise(res => {
        return (resolve = res);
      });
      const observer = new MutationObserver(records => {
        for (const record of records) {
          if (record.target instanceof HTMLCanvasElement) {
            resolve(record.target);
          }
        }
      });
      observer.observe(document);
      return promise;
    })
    .wait();
  ```
+ 等待表单可用后（在视图内、visible=true、enabled=true、两次重绘间外边框不发生变化）后填写
  ```
  await page.locator('input').fill('value');
  ```

## 请求拦截器

可以控制、修改页面中的请求。

示例：中止所有的图片请求
```
import puppeteer from 'puppeteer';

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.setRequestInterception(true);
  page.on('request', interceptedRequest => {
    if (interceptedRequest.isInterceptResolutionHandled()) return;
    if (
      interceptedRequest.url().endsWith('.png') ||
      interceptedRequest.url().endsWith('.jpg')
    )
      interceptedRequest.abort();
    else interceptedRequest.continue();
  });
  await page.goto('https://example.com');
  await browser.close();
})();
```

### 拦截器模式

+ 传统模式

传统模式下只能使用一个拦截器，如果已经调用过其中一个拦截器，再调用后续拦截器中的`abort`、`continue`、`respond`方法时会报错。

+ 协作模式

协作模式下需要给`abort`、`continue`、`respond`方法传递优先级参数，如：
```
request.continue({}, 0);
```
Puppeteer按照拦截器的注册顺序依次执行，但是只有优先级最高的拦截器会最终生效。如果多个拦截器的优先级一样，则`abort` > `respond` > `continue`。

## 调试

因为涉及到网络请求、Web API等浏览器的不同组件，在Puppeteer中调试可能是一项艰巨的任务，为此Puppeteer提供了多种调试方法。

+ 关闭无头模式

```
const browser = await puppeteer.launch({headless: false});
```
+ slow-mo
Puppeteer提供了slowMo参数降低操作速度，需要和`headless: false`配合使用。
```
const browser = await puppeteer.launch({
  headless: false,
  slowMo: 250, // slow down by 250ms
});
```
+ 捕获控制台输出
由于页面代码运行在浏览器中，因此页面代码中的控制台输出不会重定向到Node.js。但是可以通过监听`console`事件的方式输出浏览器控制台的内容。

```
page.on('console', msg => console.log('PAGE LOG:', msg.text()));

await page.evaluate(() => console.log(`url is ${location.href}`));
```

## reference
+ https://pptr.dev/
+ https://devdocs.io/puppeteer/
+ https://developer.chrome.com/docs/chromium/new-headless?hl=zh-cn
+ https://developer.chrome.com/blog/chrome-headless-shell
+ https://pptr.dev/guides/query-selectors
+ https://pptr.dev/guides/locators
+ https://pptr.dev/guides/request-interception
+ https://pptr.nodejs.cn/guides/debugging
