# TypeScript 类型定义学习指南

TypeScript 的类型系统，主要用于约束变量、描述对象结构、定义函数参数和返回值，让代码在编译阶段发现错误。

## 1. 常用基础类型

```
let name: string = 'Tom'
let age: number = 18
let isAdmin: boolean = true

let ids: number[] = [1, 2, 3]
let value: any = 'hello'
let data: unknown = 'hello'
```

| 类型                   | 含义                  |
| -------------------- | ------------------- |
| `string`             | 字符串                 |
| `number`             | 数字                  |
| `boolean`            | 布尔值                 |
| `number[]`           | 数字数组                |
| `any`                | 任意类型，基本放弃类型检查       |
| `unknown`            | 未知类型，使用前需要进行类型检查    |
| `void`               | 通常表示函数没有有意义的返回值     |
| `never`              | 不可能正常产生的值，例如必定抛错的函数 |
| `null` / `undefined` | 空值或未定义值             |

推荐：优先使用明确的类型，尽量少用 `any`。

## 2. 常用类型定义关键字

`type`

定义类型别名

```
type ID = string | number
type User = { name: string; age: number }
```

适合定义联合类型、对象类型、函数类型等。

`interface`

定义对象结构

```
interface User {
  name: string
  age: number
}
```

常用于对象、类的契约；支持继承和声明合并。

`enum`

定义枚举

```
enum Status {
  Pending,
  Success,
  Failed
}
```

默认数字从 0 开始递增，也可以指定字符串值。现代项目也常用字符串联合类型替代。

`keyof`

获取类型的键

```
type User = { name: string; age: number }
type UserKey = keyof User
// 'name' | 'age'
```

`typeof`

获取变量的类型

```
const user = { name: 'Tom', age: 18 }
type User = typeof user
```

在类型位置中使用时，提取变量的静态类型。

## 3. 对象类型：必会

```
interface User {
  id: number
  name: string
  age?: number
  readonly token: string
}
```

| 写法               | 含义                 |
| ---------------- | ------------------ |
| `id: number`     | 必填属性               |
| `age?: number`   | 可选属性               |
| `readonly token` | 只读属性，不能通过该类型直接重新赋值 |
| `User`           | 对象类型名称             |

使用：

```
const user: User = {
  id: 1,
  name: 'Tom',
  token: 'abc'
}
```

`age` 可以不填写，因为它带有 `?`。

## 4. 联合类型与交叉类型

这是 TypeScript 中非常重要的两种类型组合方式。

### 联合类型 `|`

表示多个类型中满足一个即可。

```
type ID = string | number

let id: ID = 100
id = '100'
```

常用于状态、参数和接口返回值。

### 交叉类型 `&`

表示同时具备多个类型的要求。

```
type Person = {
  name: string
}

type Employee = {
  employeeId: number
}

type Staff = Person & Employee

const staff: Staff = {
  name: 'Tom',
  employeeId: 1001
}
```

记忆：

- `A | B`：A 或 B。
- `A & B`：同时满足 A 和 B。

## 5. 函数类型定义

```
type Add = (a: number, b: number) => number

const add: Add = (a, b) => a + b
```

解释：

- `a: number`：参数类型。
- `=> number`：返回值类型。
- `Add`：整个函数的类型别名。

异步函数也可以定义：

```
async function getUser(): Promise<User> {
  return { id: 1, name: 'Tom', token: 'abc' }
}
```

`Promise<User>` 表示异步操作最终得到 `User` 类型的数据。

## 6. 泛型 `<T>`：编写通用代码

泛型允许函数或类型接收不同的类型，同时保留类型检查能力。

```
function identity<T>(value: T): T {
  return value
}

identity<string>('hello')
identity<number>(123)
```

`T` 是类型参数，可以理解为一个待确定的类型。

常见写法：

```
type Result<T> = {
  data: T
  success: boolean
}

type UserResult = Result<User>
type ListResult = Result<User[]>
```

同一个 `Result` 可以用于单个用户，也可以用于用户列表。

## 7. 常用高级类型关键字

| 关键字 / 语法  | 作用               | 示例                                   |
| --------- | ---------------- | ------------------------------------ |
| `extends` | 类型约束、继承或条件判断     | `T extends string`                   |
| `infer`   | 在条件类型中推导类型       | `T extends Promise<infer U> ? U : T` |
| `in`      | 遍历类型的键，构建映射类型    | `[K in keyof T]`                     |
| `typeof`  | 获取变量的静态类型        | `type T = typeof obj`                |
| `keyof`   | 获取对象类型的键         | `keyof User`                         |
| `as`      | 类型断言，或在映射类型中重映射键 | `value as string`                    |
| `is`      | 定义类型守卫的返回类型      | `x is User`                          |

例如，`extends` 与 `infer` 可以组合使用：

```
type Unwrap<T> =
  T extends Promise<infer U> ? U : T

type A = Unwrap<Promise<string>> // string
type B = Unwrap<number>          // number
```

## 8. 常用内置工具类型

TypeScript 自带工具类型，可以基于现有类型快速生成新类型。

```
interface User {
  id: number
  name: string
  age: number
}
```

| 工具类型            | 作用       | 示例                            |
| --------------- | -------- | ----------------------------- |
| `Partial<T>`    | 所有属性变成可选 | `Partial<User>`               |
| `Required<T>`   | 所有属性变成必填 | `Required<User>`              |
| `Pick<T, K>`    | 选取部分属性   | `Pick<User, 'id' \\| 'name'>` |
| `Omit<T, K>`    | 排除部分属性   | `Omit<User, 'age'>`           |
| `Record<K, V>`  | 定义键和值的类型 | `Record<string, number>`      |
| `Readonly<T>`   | 所有属性变成只读 | `Readonly<User>`              |
| `ReturnType<T>` | 提取函数返回类型 | `ReturnType<typeof getUser>`  |
| `Parameters<T>` | 提取函数参数类型 | `Parameters<typeof getUser>`  |

例如：

```
type UserUpdate = Partial<User>
```

生成的类型相当于：

```
{
  id?: number
  name?: string
  age?: number
}
```

适合用于更新接口：只提交需要修改的字段。

## 9. 结合 React / Zustand 的实际场景

在你学习的 Next.js、React、Zustand 项目中，经常会看到这样的定义：

```
interface UserState {
  name: string
  age: number
  setName: (name: string) => void
}

const initialState: Pick<UserState, 'name' | 'age'> = {
  name: 'Tom',
  age: 18
}
```

这里涉及三个知识点：

- `interface`：定义状态结构。
- `(name: string) => void`：定义函数类型。
- `Pick`：从完整状态中提取指定属性。

这样可以明确区分状态数据、操作函数和初始化数据的类型。

```
type WithSelectors<S> = S extends { getState: () => infer T }
  ? S & { use: { [K in keyof T]: () => T[K] } }
  : never;
```

这里的 `S` 是一个泛型类型参数，代表传入的某一种类型。之所以用 `S`，是因为它是开发者自己起的名字，没有特殊含义，也可以改成 `TStore`、`StoreType` 等。

### 1. 为什么使用 `S`？

```
type WithSelectors<S> =
  S extends { getState: () => infer T }
    ? S & { use: { [K in keyof T]: () => T[K] } }
    : never;
```

可以分成三步理解：

第一步：`<S>` 接收一个类型

```
type WithSelectors<S> = ...
```

这里的 `S` 就像函数的参数，只不过它接收的是类型，而不是运行时的值。

第二步：判断 `S` 是否符合条件

```
S extends { getState: () => infer T }
```

判断 `S` 是否具有 `getState()` 方法，并从返回值推导出状态类型 `T`。

第三步：返回增强后的类型

```
S & { use: { [K in keyof T]: () => T[K] } }
```

保留原来的 `S`，再添加 `use` 属性。

### 2. 用具体例子理解

假设传入的 Store 类型是：

```
type MyStore = {
  getState: () => {
    name: string
    age: number
  }
  setState: () => void
}
```

使用：

```
type Result = WithSelectors<MyStore>
```

此时类型参数对应关系是：

| 名称  | 实际类型                            |
| --- | ------------------------------- |
| `S` | `MyStore`                       |
| `T` | `{ name: string; age: number }` |

最终 `Result` 相当于：

```
type Result = MyStore & {
  use: {
    name: () => string
    age: () => number
  }
}
```

也就是说，`S` 代表整个 Store，`T` 代表 Store 中的状态数据。

### 3. 为什么不用 `T` 作为外层参数？

其实完全可以这样写：

```
type WithSelectors<TStore> =
  TStore extends { getState: () => infer TState }
    ? TStore & {
        use: {
          [K in keyof TState]: () => TState[K]
        }
      }
    : never
```

结果完全一样，只是变量名称更容易理解。

记住：

- `S`：整个 Store 的类型。
- `T`：从 `getState()` 推导出来的状态类型。
- `K`：遍历状态对象时的每一个属性名。

这些字母都不是 TypeScript 的保留关键字，只是常见的命名习惯。

## 10. 建议的学习顺序

1. 基础类型：`string`、`number`、`boolean`、数组。
2. 对象定义：`type`、`interface`、可选属性 `?`、只读属性 `readonly`。
3. 组合类型：联合类型 `|`、交叉类型 `&`。
4. 函数类型与泛型：`=>`、`<T>`、`extends`。
5. 类型运算：`keyof`、`typeof`、`infer`、`in`。
6. 工具类型：`Partial`、`Pick`、`Omit`、`Record`。

最重要的学习思路： 先学会定义数据结构，再学习如何组合、提取和推导类型。理解这两步之后，阅读 Zustand、Next.js 等项目中的复杂 TypeScript 类型就会容易很多。