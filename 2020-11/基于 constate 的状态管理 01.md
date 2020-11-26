# 基于 constate 的状态管理

## 历史难题 

回想了下 React 的状态管理体系，从 `Flux` 架构提出开始，出现了  `Redux`  和它的无数中间件，再到 `Saga` 这个庞然大物（光看 API 就能让你怀疑人生）。再看看实际项目的代码，各种 `depatch` 漫天，数据流看来是清晰了，可视化工具里能愉快回溯了，但是大脑和鼠标手就饱受摧残。

总体来说，技术门槛个人觉得已经超过了 Angular， 同时对于单一状态树，我也一直保持怀疑。可以说这些问题导致我对 React 一直保持了很久的观望。

好在除了 `Flux`，还出现了像 `mobx`、`rx-react` 这些成熟的替代方案，打破了官方独大的技术方向。

接触到 mobx 应该是 2018 年左右，知乎已经有了相关的讨论。当时来看，基于 `class-component` 和 `decorator` 的风格足够简单清晰，可响应数据比 `setState` 更直接，有点 `jsx` + `vue` 的感觉。于是开始在几个边缘项目中尝试，基于基础业务 (CRUD) 做了数据层和组件的封装，整体效率和体验应该说是比 vue 好的，摆脱模版对 TS 的极度不友好，也最终促使我投入 React 的怀抱。

> React 16.8 之后，`mobx` 也发布了基于 `hooks` 和 `functional-component` 重构的 `mobx-react-lite` 来替代原先的 `mobx-react`

## Hooks 时代的状态管理

随着 React 16.8 的到来，`hooks`  带领 `functional-component` 向着全能的方向发展，自然状态管理也不例外，于是出现了基于 hooks 的管理方案，如 `unstated-next`、`constate` 和之后官方背书的 `Recoil`。

Recoil 目前还处于 `0.1.2`，所有 `atom` 都挂在到一个全局状态树上（有点 Redux 的感觉）， 这也导致状态树上的数据在大型项目中会出现无法释放的问题，计划是通过类似内存的管理机制来解决这个问题 [issue](https://github.com/facebookexperimental/Recoil/issues/56) ，目前暂无进展。

unstated-next 近期维护频率也不高，原理也和 constate 相似，所以这里也不多做介绍。

那么重点来了，我们先看看 constate `1.3.0` 的核心代码感受下：[source](https://github.com/diegohaz/constate/blob/v1.3.0/src/index.tsx)


```tsx
function createUseContext(context: React.Context<any>): any {
  return () => {
    const value = React.useContext(context);
    return value;
  };
}

function constate<P, V, S extends Array<SplitValueFunction<V>>>(
  useValue: (props: P) => V,
  ...splitValues: S
): ContextHookReturn<P, V, S> {
  const Context = React.createContext(NO_PROVIDER as V);

  const Provider: React.FunctionComponent<P> = props => {
    const value = useValue(props);
    const [createMemoDeps] = splitValues;
    const deps = createMemoDeps && createMemoDeps(value);

    ...

    // deps won't change between renders
    const memoizedValue = Array.isArray(deps)
      ? React.useMemo(() => value, deps)
      : value;

    return (
      <Context.Provider value={memoizedValue}>
        {props.children}
      </Context.Provider>
    );
  };

  if (useValue.name) {
    Context.displayName = `${useValue.name}.Context`;
    Provider.displayName = `${useValue.name}.Provider`;
  }

  // const useCounterContext = constate(...)
  const useContext = createUseContext(Context);

  // const { Context, Provider } = constate(...)
  useContext.Context = Context;
  useContext.Provider = Provider;

  if (!splitValues.length) {
    // const [Provider, useCounterContext] = constate(...);
    useContext[0] = Provider;
    useContext[1] = createUseContext(Context);
    return useContext;
  }

  ...
}
```

- useValue 是一个自定义 hook 方法，在 Provider 内初始化后，将 hooks 对象放入 Context.Provider 实例中，Provider 作为 Context 的容器导出
- Context 被 createUseContext 包装到闭包方法中，内容用 `useContext` 导出供消费端调用



具体使用

```jsx
// 1. 用 constate 包装自定义 hooks
const [CounterProvider, useCounterContext] = constate(() => {
  const [count, setCount] = useState(0);
  const increment = () => setCount(prevCount => prevCount + 1);
  return { count, increment };
});

function Button() {
  // 通过 useCounterContext 消费 increment
  const { increment } = useCounterContext();
  return <button onClick={increment}>+</button>;
}

function Count() {
// 通过 useCounterContext 消费 count
  const { count } = useCounterContext();
  return <span>{count}</span>;
}

function App() {
  // 外部用状态容器包裹
  return (
    <CounterProvider>
      <Count />
      <Button />
    </CounterProvider>
  );
}

```



整个源码只有100行，却开启了 hooks 的一个新世界

- 通过 Context API 和 hooks 的结合，使得 hooks 具备了状态共享的能力

- 结合了 Context API 和 useContext，导出产物就能直接使用，让代码风格更加精简

- 更重要的一点，Context.Provider 本身是完全匹配组件的生命周期的，使得业务数据和你的业务模块能够同时初始化、销毁，完全不需要担心性能问题，简直 Amazing！

  

Talk is cheap，下一节我会用结合通用的 CRUD 场景来展示 constate 的实战用法

 [基于 constate 的状态管理 - 实践](https://github.com/gouflv/blog/blob/master/2020-11/%E5%9F%BA%E4%BA%8E%20constate%20%E7%9A%84%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86%2002%20-%20%E5%AE%9E%E8%B7%B5.md)  👈



## 参考

- [Flux](https://facebook.github.io/flux/)
- [Redux](https://redux.js.org/)
- [Redux Toolkit](https://redux-toolkit.js.org/)
- [Recoil](https://recoiljs.org/)
- [constate](https://github.com/diegohaz/constate)

