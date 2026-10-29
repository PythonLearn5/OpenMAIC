# React 常用 Hooks 速查

Hook 是 React 提供的函数，让函数组件能够使用状态、生命周期相关能力、缓存计算结果等。

## 1. 常用 Hook

| Hook          | 作用              | 典型场景          |
| ------------- | --------------- | ------------- |
| `useState`    | 管理组件状态          | 输入框、弹窗开关、计数器  |
| `useEffect`   | 处理副作用           | 请求数据、订阅事件、定时器 |
| `useCallback` | 缓存函数引用          | 减少子组件不必要的重复渲染 |
| `useMemo`     | 缓存计算结果          | 避免重复执行昂贵计算    |
| `useRef`      | 保存可变值，不因修改而触发渲染 | DOM 引用、定时器 ID |
| `useContext`  | 读取跨组件共享的数据      | 主题、用户信息、语言设置  |
| `useReducer`  | 集中管理复杂状态更新      | 多步骤表单、复杂交互    |
| `useId`       | 生成稳定的唯一 ID      | 表单标签与输入框关联    |

## 2. 重点理解：useState

用于保存状态。调用更新函数后，React 会安排组件重新渲染。

```
const [count, setCount] = useState(0);

<button onClick={() => setCount(count + 1)}>
  {count}
</button>
```

- `count`：当前状态。
- `setCount`：更新状态。
- `0`：初始值。

## 3. useEffect：处理副作用

用于组件与外部系统同步，例如请求数据、监听事件、建立连接。

```
useEffect(() => {
  console.log("组件已挂载");

  return () => {
    console.log("清理副作用");
  };
}, []);
```

依赖数组的常见含义：

| 写法                          | 执行时机                            |
| --------------------------- | ------------------------------- |
| `useEffect(() => {})`       | 每次提交渲染后                         |
| `useEffect(() => {}, [])`   | 初次挂载后；开发环境 Strict Mode 可能额外执行一次 |
| `useEffect(() => {}, [id])` | 初次挂载后，以及 `id` 变化后               |

注意：请求数据不一定必须放在 `useEffect` 中。Next.js 还支持服务端数据获取等方式。

## 4. useCallback：缓存函数

```
const handleClick = useCallback(() => {
  console.log(id);
}, [id]);
```

含义：当依赖项 `id` 没有变化时，React 通常会返回同一个函数引用。

适用场景： 将回调传给使用 `React.memo` 的子组件，或其他依赖函数引用稳定性的场景。

## 5. useMemo：缓存计算结果

```
const total = useMemo(() => {
  return products.reduce((sum, item) => sum + item.price, 0);
}, [products]);
```

含义：`products` 没有变化时，React 可以复用之前计算的 `total`。

适用场景： 过滤大量数据、复杂计算、需要稳定对象引用的场景。

## 6. useCallback 和 useMemo 的区别

### useCallback

缓存函数引用。

```
const fn = useCallback(() => doSomething(id), [id]);
```

### useMemo

缓存计算结果。

```
const value = useMemo(() => calculate(id), [id]);
```

记忆方法：Callback 缓存函数，Memo 缓存值。

两者都是性能优化工具，不是所有函数和变量都需要缓存。

## 7. useRef：保存引用

```
const inputRef = useRef<HTMLInputElement>(null);

<input ref={inputRef} />

<button onClick={() => inputRef.current?.focus()}>
  聚焦输入框
</button>
```

`useRef` 的 `.current` 可以修改，但修改本身不会触发组件重新渲染。

## 8. useContext：跨组件共享数据

```
const theme = useContext(ThemeContext);
```

适合主题、语言、用户信息等需要被多层组件访问的数据，避免逐层传递 Props。

## 9. useReducer：复杂状态管理

```
const [state, dispatch] = useReducer(reducer, initialState);

dispatch({ type: "increment" });
```

适用于多个状态之间存在关联、更新规则较复杂的场景。

## 10. 学习顺序建议

1. 基础： `useState`、`useEffect`
2. 性能优化： `useMemo`、`useCallback`
3. 引用与共享： `useRef`、`useContext`
4. 复杂状态： `useReducer`
5. 自定义 Hook： 把可复用的状态逻辑封装成自己的 Hook，例如 `useAuth()`、`useFetchData()`。

最后记住三个原则：

- Hook 只能在函数组件或自定义 Hook 的顶层调用，不能放在条件、循环或普通函数中。
- 依赖数组必须包含 Effect 或缓存函数、计算中使用的相关响应式值。
- 不要为了使用 Hook 而使用 Hook；先保证代码正确，再针对实际性能问题进行优化。