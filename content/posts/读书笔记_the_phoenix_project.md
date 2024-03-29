---
title: "不能想象没有书的世界 | The Phoenix Project"
date: 2021-08-15
tags: ["读书笔记"]
draft: false
---

![](https://img.gejiba.com/images/08d2c675e64b22061e515d5b33a96e48.jpg)

这是一本小说，讲述 IT 运维进化为 DevOps 并帮助公司取得商业成功的“传奇”故事。

我看的是英文原版，厚厚的一本，刚开始啃起来还有点费劲，有很多不熟悉的生词，在不影响理解上下文的情况下，我都直接跳过了。

说回书中的内容，故事的主角，Bill，有些意外地成为了运维部门的 VP，新官上任，就将面临巨大的考验。混乱的项目管理，各部门之间的利益纠葛，业务步步紧逼的压力，都是 Bill 需要翻越的大山。

幸运的是，Bill 遇到了一位优秀的导师，Eric。Eric 指点 Bill 使用三步工作法梳理工作流程，将日常工作划分为四类，分析并解决工作流中的瓶颈，思考如何更快地帮助业务前进，然后逐渐形成了 DevOps 的文化。

相比较于纯粹讲述方法论或者解决方案的专业书，这本以小说的形式，生动地描述了 IT 行业内会遇到的种种难题，以第一人称的视角来思考，结合方法论来解决问题，更有带入感。

书中有两点令我影响深刻。首先是 Eric 这个角色，为 Bill 指点迷津，一步一步地帮助 Bill 思考和解决问题（所以，为什么不让 Eric 直接领导整个 IT 部门？），不得不感叹，这就是生命中的贵人吧，如果有缘遇到这样的高人，可一定要多多请教。同样地，也不能吝惜自己掌握的知识，有机会也要多帮助他人。第二点是关于信任，公司的 CEO，Steve 在动员大家全力投入凤凰项目时，轮流让高管、组员讲述个人经历，拉近相互的距离，能让大家产生信任感，“Solving complex business problem requires teamwork, and teamwork requires trust”, that's right。

另外书中有涉及到的一些方法论，摘录在下方，方便在日后的工作过程中，回顾和思考。

## 一张图

![](https://img.gejiba.com/images/912d13829191e8348199fba412da41e9.jpg)

这张图展示了资源使用率与等待时间的关系，当资源使用率超过 80%，等待时间就会直线上升。

例如在一个单位时间（1 小时）内，你有 90% 的时间在忙碌，依赖你的等待时间就会是 0.9 / 0.1 = 900%，也就是 9 小时，会严重阻塞工作流。

## 三步工作法

第一步是构建从左到右（开发 -> 运维 -> 客户）的工作流，使用 kanban 等工具将流程可视化，通过持续交付（构建、集成、部署）来优化、趋近整体目标。

第二步是沿着从右到左的价值流，持续地快速反馈，将利益最大化，防止问题重复发生或快速修复，这样就能从源头上保证质量，沉淀领域知识。

第三步是创造公司文化：勇于尝试和重复练习。尝试需要承担风险，并能从成败中吸取经验教训，而一旦出现问题，重复练习带来的成熟度，能让项目回退到安全地带，恢复正常运作。

## 四种工作类型

1. 业务项目
2. 内部项目（基础架构、运维工具和平台等）
3. 变更（通常由 1、2 引起）
4. 计划外的工作（生产事故等救火工作）

