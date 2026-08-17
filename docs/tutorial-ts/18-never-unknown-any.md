# 第 18 章 · `never` / `unknown` / `any` 的三分法

## 本节要学的概念

- `any`:关掉类型检查,最危险,代表「失去类型安全」。
- `unknown`:安全的「未知」,用它必须先收窄。
- `never`:代表「不可能发生」,是穷尽检查与死代码的核心。
- 项目对三者的使用纪律。

## 为什么它们容易混

`any`、`unknown`、`never` 都叫「特殊类型」,但语义南辕北辙。新手最常见的问题是把它们当成「差不多」的东西。其实它们是三个**完全不同的工具**,对应三种不同的场景。

## `any`:关掉检查的开关

```ts
let x: any = 42
x = "hello"
x.foo.bar.baz()   // 全都不报错
x()               // 也不报错
```

`any` 的意思是「**这个值是任意类型,别检查了**」。它有两个互相矛盾的性质:

- **可用性极强**:能赋值给任何类型,也能从任何类型赋值给它,能调用任何方法。
- **安全性清零**:一旦 `any` 出现,类型检查从这里开始「失明」,错误会像水一样从 `any` 这个口子漏出去。

所以 `any` 的代价是「**局部关掉类型系统**」。用 `any` 等于对编译器说「这里我不需要你」。

## `unknown`:安全的「未知」

```ts
let y: unknown = 42
y = "hello"        // 合法,unknown 也可以装任何值
y.foo              // 报错!unknown 上不能直接访问属性
y.toUpperCase()    // 报错!必须先收窄
```

`unknown` 和 `any` 一样「能装任何值」,但**保留了安全性**:在 `unknown` 上做任何操作,都必须**先收窄**(第 6 章)——用 `typeof`、类型谓词(第 16 章)等证明它是什么,才能操作。

一句话记:

> **`any` 是「我知道是什么,别管了」;`unknown` 是「我不知道是什么,得先确认」。**

处理「外部输入」——模型输出、网络响应、文件内容——的正确类型就是 `unknown`。第 16 章那些 `isPlainJsonRecord(value: unknown)` 守卫函数,参数都是 `unknown`:先收窄成确定类型,再安全操作。

## `never`:代表「不可能」

```ts
function fail(): never {
  throw new Error("x")
}

function loop(): never {
  while (true) {}
}
```

`never` 表示「**这个位置的值不会发生**」——一个永远不会正常返回的函数,返回类型是 `never`;一个穷尽可能的分支里,剩余的类型是 `never`。

`never` 有两条你在前几章已经用过的性质:

1. **`never` 是「一切类型的子类型」**,可以赋给任何类型(所以第 7 章 `assertNever(x: never)` 能接受任何「已穷尽」的值)。
2. **在联合里,`never` 会被自动忽略**。`string | never` 化简为 `string`——这正是第 13、15 章「映射成 never 然后索引访问取联合」的过滤原理:把不想要的键映射成 `never`,联合里就自动消失了。

第 7 章的穷尽检查,本质就是:**走到 `default` 分支时,`cause` 的类型应是 `never`;如果它不是 `never`(说明有新情况没处理),`assertNever` 就编译报错。** 用 `never` 的「不可能」含义,反向探测「你漏了 case」。

## 项目里的纪律:把 any「钉」回 unknown

项目对 `any` 的态度非常严格,看一个真实例子。 [`packages/util/timeout/src/index.ts`](../../packages/util/timeout/src/index.ts) 里有一句注释:

```ts
// AbortSignal.reason is typed `any`; pin it to `unknown` so no `any` leaks
// and consumers must narrow before using it.
const reason: unknown = x.reason
```

这段注释精确表达了项目的立场:

- `AbortSignal.reason` 这个属性,在 TS 标准库里被声明成了 `any`(历史包袱)。
- 项目不想让这个 `any` 泄漏进自己的代码,于是**手动把它「钉回」`unknown`**:`const reason: unknown = x.reason`。
- 效果是:`any` 到此为止,下游要用 `reason` 必须先收窄。

这就是「**任何 `any` 都要解释或约束**」的体现——根 AGENTS.md 写得很清楚:「一切在 `strict: true` + `noImplicitAny` 下编译;每处残留的 `any` 都要解释为什么无法收窄」。项目宁可多用几个 `unknown` + 守卫函数,也不让 `any` 漂进来。

## 一张对照表

| | 能被赋任何值 | 能赋给任何类型 | 能直接操作(调方法/取属性) | 典型用途 |
|---|---|---|---|---|
| `any` | ✅ | ✅ | ✅(危险) | 应尽量避免 |
| `unknown` | ✅ | ❌ | ❌(先收窄) | 外部输入的安全入口 |
| `never` | ❌ | ✅(子类型) | — | 穷尽检查、死代码、过滤 |

## 你现在能自己读了

1. 回到第 7 章的 `describe(cause)` 例子,重新体会 `assertNever` 的参数为什么是 `never`,它如何靠 `never` 实现「漏 case 就报错」。

2. 在 [`packages/core/tools/src/json-schema.ts`](../../packages/core/tools/src/json-schema.ts) 里,观察那些 `isXxx(value: unknown): value is Yyy` 守卫函数——它们是 `unknown` → 收窄 → 确定类型的完整示范。

3. 想想第 15 章 `dispatch.ts` 里那几处 `as`:作者为什么在这里用 `as`(而不是 `any` 或 `unknown`)?它们和「钉回 unknown」的立场有何关联?

## 小结

- `any` 关掉检查,最危险;`unknown` 是安全未知,须先收窄;`never` 是「不可能」。
- `never` 用于穷尽检查,且在联合里自动被忽略(过滤原理)。
- 项目严格管控 `any`:外部边界用 `unknown` + 守卫,残余 `any` 必须钉回 `unknown` 或解释。

## 思考题

1. 用一句话说清 `any` 和 `unknown` 的关键区别(提示:安全性/能否直接操作)。
2. `string | never` 结果是什么?为什么这个性质能让「映射成 never + 索引访问」实现过滤?
3. 为什么「把 `AbortSignal.reason` 钉回 unknown」是对的做法?如果你直接把它当 `any` 传播出去,风险在哪里?

---

上一章:[第 17 章 · 名义类型:Branded 品牌类型](17-branded-types.md)
下一章:[第 19 章 · 项目的类型规范速览](19-project-type-conventions.md)
