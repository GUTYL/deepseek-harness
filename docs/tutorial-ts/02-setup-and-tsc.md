# 第 2 章 · 环境与最小 TypeScript 程序:编译与 `strict`

## 本节要学的概念

- 最小 TS 程序的写法,以及它是怎么被「编译」成 JS 的。
- `tsconfig.json` 是什么,`strict` 开关做什么。
- `import type` 与普通 `import` 的区别——一个贯穿本项目的习惯用法。

## 一个最小的 TS 程序

新建一个文件 `hello.ts`:

```ts
const name: string = "DeepSeek"
console.log("Hello, " + name)
```

对比一下你熟悉的 JS,唯一的区别是 `name` 后面多了 `: string`。这叫做**类型标注(type annotation)**:它告诉 TypeScript「`name` 是字符串类型」。

现在用编译器 `tsc` 处理它:

```sh
tsc hello.ts
```

会生成一个 `hello.js`,你打开看:

```js
const name = "DeepSeek"
console.log("Hello, " + name)
```

` : string` 消失了。这正是第 1 章说的:**类型标注只存在于编译期,编译成 JS 后被擦除(erased)**。运行 `hello.js` 和运行你手写的 JS 完全一样,类型没有带来任何运行期成本。

## `tsconfig.json` 与 `strict`

真实项目不会一个个 `tsc` 文件,而是用一个配置文件 `tsconfig.json` 告诉编译器「怎么编译整个项目」。

打开本项目的 [`tsconfig.base.json`](../../tsconfig.base.json),它是所有包的公共编译配置。挑出最关键的一行:

```json
"strict": true
```

`strict` 是一个「总开关」,它一次性打开一批严格检查。简单说:

- **不严格**的 TS:能推断就推断,推断不出的地方悄悄放宽成 `any`(第 18 章会讲 `any` 是什么,以及为什么它危险)。
- **严格**的 TS:凡是可能出错的地方,都逼你显式写清楚类型。

为什么这件事重要?因为「TS 能不能救你」,完全取决于 `strict` 开没开。关掉 `strict` 的 TS 类型检查几乎形同虚设;而本项目把它,连同下面这些「更严格的兄弟开关」,全部打开了:

```json
"strict": true,
"noUncheckedIndexedAccess": true,
"exactOptionalPropertyTypes": true,
"noImplicitOverride": true
```

这些「兄弟开关」会在后面章节结合真实代码解释(比如第 3 章的 `noUncheckedIndexedAccess`)。现在你只需要知道:**它们让项目里的类型写得非常「诚实」——这也是我们拿它当教材的原因之一**。

> 本节只讲概念,不需要你亲手搭环境。学习路径见[开发指南](../development.md);本项目需要 Node `^22.19 || >=24`,类型检查命令是 `pnpm run typecheck`。

## 编译 vs 运行:两个世界的边界

第 1 章说过「类型编译后消失」。这里再往深走一步,因为这个概念会解释后面很多「看起来奇怪」的写法。

TypeScript 代码里,**有两类东西**:

| | 编译期(给编译器看) | 运行期(真正执行) |
|---|---|---|
| 例如 | 类型标注 `: string`、`interface`、`type` | 变量、函数、表达式 |
| 编译成 JS 后 | **全部消失** | **保留** |

现在看一个本项目里无处不在的写法:

```ts
import type { Context } from '@deepseek-ai/cordis'
```

注意 `import` 后面多了个 `type`。它的含义是:

- 普通 `import`:我要在运行期真的**用**这个模块(调用它的函数、读它的值)。
- `import type`:我**只在类型标注里提到**这个模块,运行时不碰它。

因为「只在类型里用」的东西编译后就没了,所以 `import type` 编译后**不会生成任何真实的 import 语句**——也就是不增加任何运行期依赖。

什么时候用哪个,规则很简单:**如果一个 import 进来的东西,只出现在 `:` 后面的类型位置,就用 `import type`;如果它出现在 `=` 右边的运行位置,就用普通 `import`。** 本项目贯彻得很彻底,你会在每个文件顶部看到这两种 import 混用。

## 你现在能自己读了

打开本项目的 [`tsconfig.base.json`](../../tsconfig.base.json),做两件事:

1. 找到 `"strict": true`,确认它开着。
2. 找到 `"paths"` 这一大段(文件名到文件路径的映射)。你现在不需要理解它,只需要知道:**这就是告诉编译器「`@deepseek-ai/dsh-xxx` 这个包名,实际文件在磁盘的哪里」的映射表**。后面读代码时,你看到 `import ... from '@deepseek-ai/dsh-agent'`,知道它能找到对应源码,靠的就是这里。

再打开任意一个源码文件,比如 [`packages/core/agent/src/dispatch.ts`](../../packages/core/agent/src/dispatch.ts) 顶部,观察前几行 import 的写法——哪些是 `import type`,哪些是普通 `import`。

## 小结

- 类型标注 `: string` 编译后被擦除,不产生运行成本。
- `tsconfig.json` + `"strict": true` 是类型检查「真正生效」的前提;本项目把严格开关全开了。
- `import type` 只在类型位置引用模块,编译后不生成真实依赖;只在 `:` 后使用就用它。

## 思考题

1. `import type { Context }` 和 `import { Context }` 编译出来的 JS 有什么不同?为什么项目对前者如此执着?
2. 一个团队如果 `strict: false` 但声称「我们用了 TS」,你该怎么反驳?提示:结合「类型检查形同虚设」。
3. 在 [`tsconfig.base.json`](../../tsconfig.base.json) 里找到 `noUncheckedIndexedAccess`,猜猜这个开关管什么(第 3 章会揭晓)。

---

上一章:[第 1 章 · 为什么要学 TypeScript](01-why-typescript.md)
下一章:[第 3 章 · 变量与基础类型注解](03-basic-types.md)
