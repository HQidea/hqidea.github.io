---
title: ComfyUI
date: 2024-04-09 20:23:15
tags:
---
ComfyUI是一个强大的模块化stable diffusion图形化界面和后端服务，提供了通过workflow的方式设计stable diffusion工作流。相比[WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui)，ComfyUI提供了灵活的方式自定义工作流，支持工作流复用，或者修改、替换甚至是开发其中的节点来满足不同的需求。ComfyUI能够处理更复杂、更大规模的任务，同时内部对性能做了优化，生成更快，占用显存更小。

## 概念

+ 潜在空间: 机器学习中用于表示数据被压缩或编码成较低维度的特征空间
+ CLIP（Contrastive Language Image Pre-training，对比语言图像预训练）：是一个常用的Text Encoder模型，将文本embedding成特征向量
+ MODEL：具备对图片进行降噪能力的模型，一般有2种格式：
  + safetensors：仅包含模型的参数，加载安全性好，速度快
  + ckpt：除了模型参数外，还包含训练过程中的优化器状态，相当于训练快照，可以用于恢复训练，但是体积较大且存在安全风险
+ VAE（Variational Auto Encoder，变分自编码器）：将图片编码成潜在空间以及将潜在空间解码成图片
+ LoRA(Low-Rank Adaptation of Large Language Models, 大语言模型的低秩自适应)：最初为大语言模型设计，在不修改大模型各层参数的前提下，在各层之间注入可训练的层，类似在两个函数之间做参数转换，大大降低了大模型微调的训练量，后来也应用到Stable diffusion模型中

## 原理

### 文生图

1. 用户输入的prompt会被Text Encoder embedding成特征向量
2. 特征向量和一张随机噪声图（如电子雪花图）一起放入Image Information Creator中，Creator将特征向量和噪声图转换到一个潜在空间内，然后根据特征向量将噪声图进行降噪
3. 降噪后的结果会被Image Decoder解码成一张图片

![alt text](image.png)

> 整个过程，与其说 AI 是在「生成」图片，不如称其为「雕刻」更合适。就像米开朗基罗在完成大卫雕像后，说过的一句话那样：雕像本来就在石头里，我只是把不要的部分去掉。

Image Information Creator做的事情就是将一张充满噪点的图片去除其中不需要的噪点。

### 图生图

图生图主要有2种方式，分别是重绘和参考。

#### 重绘

重绘使用用户输入的图片作为基底，在其之上生成新的图片。原理和文生图基本一致，只是将随机噪声图替换成了用户输入的图片，然后通过模型增加噪声。

#### 参考

参考是将图片作为prompt输入，模型根据prompt的指令生成新的图片。原理也和文生图基本一致，只是在第一步中除了输入文本prompt，还需要输入一张图片，使用相应的模型将文本和图片同时embedding成特征向量。

## GUI

理解了上面的概念和原理之后，就很容易看懂ComfyUI的界面了：
![alt text](image-1.png)
还是有些疑惑？那我们把界面整理一下：
![alt text](image-3.png)
这样是不是就明白了

GUI中还有2个在原理中未涉及到的节点：
+ Load Checkpoint：可以理解为模型加载器，加载了MODEL、CLIP、VAE 3个模型
+ Save Image：保存图片

再对Text Encoder做一些补充说明：

Text Encoder分为正向prompt和负向prompt，在正向prompt列出你希望在结果图片中看见的风格或内容，在负向prompt列出你不想在结果图片中看见的风格或内容。一般来说，编写prompt有以下几个原则：
1. prompt不是越长越好，尽量控制在75个token（约60个单词）以内
2. 不需要写完整的句子，只需要列出关键词，关键词之间使用逗号分割
3. 越重要的关键词放在越前面
4. 可以使用(keyword:weight)的方式来控制关键词的权重，如(cute:1.2)表示增加.2倍的效果，如果weight小于1则会减少效果

## reference
+ https://github.com/comfyanonymous/ComfyUI
+ https://comfyui.jpanda.cn/
+ https://www.comflowy.com/zh-CN/basics/stable-diffusion-foundation
+ https://zhuanlan.zhihu.com/p/619247417
