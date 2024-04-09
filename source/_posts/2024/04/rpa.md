---
title: RPA
date: 2024-04-09 15:49:27
tags:
---
RPA全称Robotic Process Automation，中文名机器人流程自动化，使用机器模拟人类的操作，执行各种重复、流程化的任务。

## 发展历史

+ 第一阶段：辅助人类完成特定的任务，比如Excel中的宏，整个过程需要人工干预
+ 第二阶段：可以独立地完成业务流程中的部分工作，但是需要人工的控制和管理
+ 第三阶段：使用一些基础的感知能力，自动化地完成一些复杂的操作，比如写邮件、开发票等
+ 第四阶段：利用人工智能技术，包括机器学习和自然语言处理等算法实现理解业务、自主决策甚至优化任务执行步骤等

## 应用领域

+ 客服自动化：通过结合AI和RPA可以实现自动回复用户咨询、快速处理订单等操作
+ 财务流程自动化：结合OCR技术可以实现自动识别提取发票、自动生成报表等操作
+ 人才招聘自动化：实现高效整理人才简历、智能邀约、面试助手等功能

## 头部厂商

+ 国外
  + UiPath：BS架构，提供直观易用的界面，并且拥有广泛的用户社区
  + BluePrism：CS架构，面向企业提供安全、可控、智能的自动化平台
  + AutomationAnywhere：CS架构，主要面向开发者并提供脚本功能

+ 国内
  + 金智维：内置大量的rpa函数和应用操作经验，提供ocr+nlp等ai能力，擅长金融领域
  + 来也：收购了UiBot，提供可视化编程、流程录制、双类型切换等功能，擅长政企

## 分类

+ 有人参与自动化：这类自动化需要和人类配合，只能完成流程中的一部分任务
+ 无人参与自动化：这类自动化可以按预先定义好的流程自动完成全部任务

## 常见组件

+ 自动化核心：负责与具体的业务系统交互，识别、控制操作目标
+ 流程设计器：负责设计RPA的执行流程，一般通过录制视频或者通过低代码workflow编排，在未来也可以由AI大模型来设计流程
+ 算法工具库：提供OCR、NLP等算法能力
+ 运维监控：负责监控管理RPA机器人的调度、执行、日志

## 自动化协议/工具

自动化核心主要有2类任务，分别是对操作目标的定位和控制。

+ 兼具定位和控制的协议/工具
  + WebDriver: 由Selenium推动的浏览器控制协议，被大多数浏览器接受，主要工具有Selenium、WebdriverIO，而[Appium](https://blog.hqhome.net/posts/appium/)将自动化的对象由浏览器扩展到了桌面和移动设备
  + CDP(Chrome Devtools Protocol)：类Chromium浏览器的调试协议，可以实现检查、调试、控制等功能，主要工具有Puppeteer
  + WebDriver-BIDI：由于WebDriver效率较低，且不支持低级别的控制；CDP只能在类Chromium浏览器中使用，因此WebDriver-BIDI吸取了前两个协议的优点，设计了一套面向未来的自动化标准，支持高效传输、低级别的控制和跨浏览器兼容
  + Win32 API: Windows操作系统的底层API
+ 额外的定位方式
  + CV：利用计算机视觉对屏幕元素的识别和定位
+ 额外的控制方式
  + 模拟鼠标、键盘的操作

## refenerce
+ https://www.woshipm.com/ai/3624725.html
+ https://zhuanlan.zhihu.com/p/489792917
+ https://zhuanlan.zhihu.com/p/86066099
+ https://juejin.cn/post/7350586974908284947
+ https://developer.chrome.com/blog/webdriver-bidi?hl=zh-cn
+ https://www.signitysolutions.com/blog/rpa-tools-comparison
+ https://www.zhihu.com/question/334568023/answer/3305586814
+ https://www.datagrand.com/blog/rpa%E7%95%8C%E9%9D%A2%E5%85%83%E7%B4%A0%E5%AE%9A%E4%BD%8D%E4%B8%8E%E6%93%8D%E6%8E%A7%E6%8A%80%E6%9C%AF%E8%AF%A6%E8%A7%A3-%E8%BE%BE%E8%A7%82%E6%95%B0%E6%8D%AE.html
