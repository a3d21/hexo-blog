---
title: 我折腾过的编程语言
date: 2022-11-13 14:40:12
tags: 编程
---

我折腾过的编程语言有

| 语言    | 特点               | 说明                                                                                     |
| ------- | ------------------ | ---------------------------------------------------------------------------------------- |
| Go      | 简单，静态编译     | 最近3年主要工作语言                                                                      |
| Java    | 生态成熟           | Go之前主要工作语言，现在偶尔写些业务项目                                                 |
| Clojure | 数据即代码，元编程 | 17年学函数式编程时学的，写过GUI程序和几个Web小项目                                       |
| Haskell | 强类型函数式       | 17年学函数式编程时学的，主要是开阔眼界                                                   |
| Kotlin  | 加糖版Java，有协程 | 写过Demo，无实际项目经验                                                                 |
| Groovy  | 动态版本Java       | 尝试过在Java项目中混合编程                                                               |
| Python  | 库成熟             | 经常用来写测试脚本。用它写过爬虫，但发布时依赖处理很麻烦，所以很少在项目中用它           |
| Scala2  | FP+OOP大成         | 提供FP的实际工程解决方案。虽然没深度使用过，但通过学习理解了Monad等概念在工程上的应用  |
| node.js | 脚本语言，对云友好 | 很多云平台支持托管node.js项目，用它写小项目很方便                                        |
| Rust    | 强类型，内存安全   | 18年学Go时了解到，并开始学习的。它完美解决了当时Go的所有痛点：依赖管理、泛型支持、FP支持 |

随着编程经验积累，我越来越发现，编程问题的实际难点在于
1. 对问题的理解与抽象
1. 对底层原理的理解与应用

相反，编程语言常常不是那么重要的。如果我理解了EventSourcing，我可以用任意一种语言实现EventSourcing架构。
如果让我从零开始实现一个系统，我会优先考虑在该领域生态更成熟的语言。如果是新兴领域，再看语言的抽象能力和领域匹配度。


但是，如果让我选出最喜欢的语言，我会毫不犹豫地选Clojure。
- Clojure是一门LISP方言，它可以像操作数据一样操作自身代码，构建代码抽象。写LISP是个思考过程，代码越写与自己的想法越契合。做个对比，如果我写Java，我需要将问题解构成`Class`和`Object`，可是有些场景（eg: 测试脚本）这套模型未必适合，所以写出的代码难以理解。但如果用LISP，可以按自己的想法任意组织代码，过程就像写文章一样。
- Clojure是一门JVM语言，这意味着它可以方便地用上Java生态，很方便。

Clojure现在工程应用很少，如果哪天我创业，它也许会成为我的“秘密武器”。