# 以 DeepSeek Harness 为教材学 TypeScript

> 本教程是一份面向**TypeScript 初学者**的中文自学材料。它以本仓库(`deepseek-harness`)的真实源码为教材,边学 TypeScript 语言、边读懂整个项目的设计与实现。它不是正式的参考文档,而是给学习者的引路地图;术语首次出现时会用括号标注英文原文。

## 这门课教什么

两条线交织推进:

- **线 A · TypeScript 语言**:从「类型是注解」这一句话开始,一路讲到泛型(generics)、条件类型(conditional types)、声明合并(declaration merging)这些项目里真正在用、也真正难读的特性。
- **线 B · 读懂项目**:每学完一批语法,就解锁读项目的一块区域。学完第五篇,你能独立读懂 agent 主循环、工具执行管道、事件系统这些核心代码。

**为什么拿一个真实项目当教材?** 因为脱离真实代码讲类型,容易讲成「语法手册」;而读真实代码时,你会看到每个语法为什么存在——它解决了一个具体的问题。本教程引用的每一段代码,都来自本仓库的 `packages/` 或 `docs/`,你可以随时点开原文件看上下文。

## 你需要什么

- 一点 JavaScript 基础——知道变量、函数、对象、`if`/`for` 长什么样即可,不需要会用 TypeScript。
- 一个能看代码的编辑器,以及本仓库一份 checkout(用来跟着翻文件)。
- 本教程**不需要** API key,也不需要跑起来项目——所有例子都是「读代码 + 思考」,不要求执行。

## 章节结构

每章固定五段:本节要学的概念 → 语法讲解(最小可运行例子)→ 项目里的真实用法 → 「现在你能自己读了」→ 小结与思考题。思考题不需要执行,也不需要标准答案,它们的作用是帮你在脑中把知识钉住。

## 学习路径要览

```text
第一篇(1~5)    TS 起步:类型是什么、基础注解、函数、接口
第二篇(6~11)   类型系统进阶:联合、判别联合、泛型、类型运算
第三篇(12~15)  TS 高级武器:条件类型、声明合并 → 读懂 dispatch.ts
第四篇(16~19)  类型守卫、品牌类型、never/unknown/any、工程规范
第五篇(20~26)  用 TS 读懂整个项目:Cordis、服务、事件、主循环
```

## 目录

### 第一篇 · TypeScript 起步与项目的「地基」

- [第 1 章  为什么要学 TypeScript,以及这个项目是什么](01-why-typescript.md)
- [第 2 章  环境与最小 TypeScript 程序:编译与 `strict`](02-setup-and-tsc.md)
- [第 3 章  变量与基础类型注解](03-basic-types.md)
- [第 4 章  函数:参数、返回值、可选与默认](04-functions.md)
- [第 5 章  对象与接口(interface)](05-interfaces.md)

### 第二篇 · 类型系统进阶

- [第 6 章  联合类型与收窄(narrowing)](06-unions-and-narrowing.md)
- [第 7 章  判别联合(discriminated union)与穷尽检查](07-discriminated-unions.md)
- [第 8 章  类型别名、数组与元组](08-type-aliases-arrays-tuples.md)
- [第 9 章  泛型(generics):写会复用的代码](09-generics.md)
- [第 10 章  类型运算:`keyof` / `typeof` / 索引访问](10-keyof-typeof-indexed.md)
- [第 11 章  映射类型与内置工具类型](11-mapped-and-utility-types.md)

### 第三篇 · TypeScript 高级武器

- [第 12 章  条件类型与 `infer`](12-conditional-types-infer.md)
- [第 13 章  模板字面量类型](13-template-literal-types.md)
- [第 14 章  声明合并与模块扩展(declaration merging)](14-declaration-merging.md)
- [第 15 章  高级综合:完整读懂 `dispatch.ts`](15-reading-dispatch.md)

### 第四篇 · 类型守卫、名义类型与工程规范

- [第 16 章  类型谓词与断言守卫(`is` / `asserts`)](16-type-predicates-asserts.md)
- [第 17 章  名义类型:`Branded<B>` 品牌类型](17-branded-types.md)
- [第 18 章  `never` / `unknown` / `any` 的三分法](18-never-unknown-any.md)
- [第 19 章  项目的类型规范速览](19-project-type-conventions.md)

### 第五篇 · 用 TypeScript 读懂整个项目

- [第 20 章  心智模型:一切皆插件(Cordis)](20-cordis-model.md)
- [第 21 章  Service 与依赖注入的真实实现](21-services.md)
- [第 22 章  事件系统与五种投递模式](22-events.md)
- [第 23 章  LLM 词汇表:消息与流](23-llm-vocabulary.md)
- [第 24 章  核心包地图与能力接缝(seam)](24-package-map-seams.md)
- [第 25 章  Agent 主循环:turn/step 状态机](25-agent-loop.md)
- [第 26 章  工具执行管道与收尾](26-tool-pipeline.md)

## 如何读这份教程

1. 按顺序读,不要跳章——每一章的知识都建立在前面章节之上。
2. 看到「项目里的真实用法」时,打开原文件对照着看,不要只看摘录。
3. 思考题先自己想,再往下读;想不出来就带着问题继续,答案会在后面章节自然浮出。
4. 读完第五篇,回到[架构文档](../architecture.md)再通读一遍——你会发现它突然变清晰了。

> 英文术语对照贯穿全文,完整术语表见项目自带的[术语表(glossary)](../glossary.md)。
