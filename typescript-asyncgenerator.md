# `AsyncGenerator` 定义在哪里

> 结论：`AsyncGenerator` 不在 cc-haha 仓库里定义，它是 **TypeScript 标准库自带的全局类型**，不需要 import。

---

## 1. 类型定义位置

TypeScript 安装目录下：

```text
node_modules/typescript/lib/lib.es2018.asyncgenerator.d.ts
```

核心定义：

```ts
interface AsyncGenerator<T = unknown, TReturn = any, TNext = any>
  extends AsyncIterator<T, TReturn, TNext> {
  next(...args: [] | [TNext]): Promise<IteratorResult<T, TReturn>>
  return(value: TReturn | PromiseLike<TReturn>): Promise<IteratorResult<T, TReturn>>
  throw(e: any): Promise<IteratorResult<T, TReturn>>
  [Symbol.asyncIterator](): AsyncGenerator<T, TReturn, TNext>
}
```

三个类型参数：

| 参数 | 含义 |
|---|---|
| `T` | `yield` 出来的值类型 |
| `TReturn` | `return` 的值类型 |
| `TNext` | `next(value)` 传入的值类型 |

所以：

```ts
AsyncGenerator<Message, void>
```

表示 `yield Message`、最终 `return void`。

## 2. 相关类型在哪个 lib

| 类型 | 所在 lib 文件 |
|---|---|
| `AsyncIterator` / `AsyncIterable` | `lib.es2018.asynciterable.d.ts` |
| `AsyncGenerator` / `AsyncGeneratorFunction` | `lib.es2018.asyncgenerator.d.ts` |
| `Symbol.asyncIterator` | `lib.es2018.asynciterable.d.ts` |
| `AsyncIterableIterator` | `lib.es2018.asynciterable.d.ts`（兼容旧写法） |

## 3. 为什么不用 import

cc-haha 的 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowJs": true,
    "jsx": "react-jsx",
    "baseUrl": ".",
    "paths": {
      "@ant/claude-for-chrome-mcp": ["./stubs/ant-claude-for-chrome-mcp.ts"],
      "color-diff-napi": ["./stubs/color-diff-napi.ts"],
      "src/*": ["./src/*"]
    },
    "types": ["bun-types"]
  }
}
```

没有显式写 `lib`，但 `target: ESNext` 会自动引入对应的 lib 链，其中包含 `lib.es2018.asyncgenerator.d.ts`。

这些是**全局 ambient 声明**，任何 `.ts` 文件都能直接使用，所以 `runAgent.ts` 里不需要：

```ts
import type { AsyncGenerator } from '...'
```

## 4. 运行时层面

运行时也没有一个用户可见的 `AsyncGenerator` 类。它由 JavaScript 引擎实现，原型链大致是：

```text
Object.prototype
  └─ %AsyncIteratorPrototype%
       └─ %AsyncGeneratorPrototype%
            └─ async function* 返回的实例
```

实例上可调用的方法：

```ts
next()                    // Promise<IteratorResult<T, TReturn>>
return(value?)            // 提前结束
throw(error)              // 向生成器内部抛错
[Symbol.asyncIterator]()  // 供 for await...of 使用
```

Bun 使用 JavaScriptCore，这部分是引擎内置行为，不经过仓库代码。

## 5. 在 cc-haha 里的使用

仓库里没有定义，只有使用。统计到 99 处，例如：

```ts
// src/tools/AgentTool/runAgent.ts:328
export async function* runAgent({
  ...
}): AsyncGenerator<Message, void> { ... }

// src/query.ts
export async function* query(...): AsyncGenerator<...> { ... }
```

`runAgent` 是一个 async generator：

- 调用它不会立刻执行函数体，只返回一个 `AsyncGenerator` 对象；
- 每次 `for await (const message of runAgent(...))` 推进一次，产出一条 `Message`；
- 调用方可以中途 `return()` 或通过 `abortController` 终止。

## 6. 容易混淆的几个类型

| 类型 | 能力 |
|---|---|
| `Iterator<T>` | 只有同步 `next()` |
| `Iterable<T>` | 只有 `[Symbol.iterator]()` |
| `AsyncIterator<T>` | 只有异步 `next()` |
| `AsyncIterable<T>` | 只有 `[Symbol.asyncIterator]()` |
| `AsyncGenerator<T, R, N>` | 同时具备 `next/return/throw/[Symbol.asyncIterator]` |
| `AsyncIterableIterator<T>` | `AsyncIterator` + `AsyncIterable` 的组合，旧式写法 |

`async function*` 的返回类型是 `AsyncGenerator`，不是 `AsyncIterableIterator`。

## 7. 想看定义本身

IDE 里 `Cmd+Click` / `Ctrl+Click` 点击 `AsyncGenerator`，会跳到：

```text
node_modules/typescript/lib/lib.es2018.asyncgenerator.d.ts
```

或者命令行：

```bash
npx -p typescript cat node_modules/typescript/lib/lib.es2018.asyncgenerator.d.ts
```

## 8. 补充：编译目标的影响

当前 `target: ESNext`，Bun 会把 `async function*` 原样保留为原生语法，不会生成降级 helper。

如果把 target 降到 `ES2015` 或更低，TypeScript 会生成 `__asyncGenerator` / `__awaiter` 之类的辅助函数（通常来自 `tslib`），这时 `AsyncGenerator` 的行为由 helper 模拟，但类型仍然是同一个 lib 声明。
