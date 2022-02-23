# 与 React 一起使用

XState 可以与 React 一起使用：

- 协调本地状态
- 高效管理全局状态
- 使用其他 Hooks 的数据

在 [Stately](https://stately.ai), 我们喜欢这个组合。 它是我们创建内部应用程序的首选方式。

要寻求帮助，请查看 [我们 Discord 社区中的 `#react-help` 频道](https://discord.gg/vedXj62MfQ)。

## 本地状态

用 [React hooks](https://reactjs.org/hooks) 是在你的组件中使用状态机的最简单方法。 您可以使用官方的 [`@xstate/react`](https://github.com/statelyai/xstate/tree/main/packages/xstate-react) 为你提供开箱即用的 hooks，例如 `useMachine `。

```js
import { useMachine } from '@xstate/react';
import { toggleMachine } from '../path/to/toggleMachine';

function Toggle() {
  const [current, send] = useMachine(toggleMachine);

  return (
    <button onClick={() => send('TOGGLE')}>
      {current.matches('inactive') ? 'Off' : 'On'}
    </button>
  );
}
```

## 全局 State/React Context

我们推荐使用 XState 和 React 管理全局状态的方法是使用 [React Context](https://reactjs.org/docs/context.html)。

> 'context' 有两个版本：XState 的 [context](../guides/context.md) 和 React 的 context。 这有点令人困惑！

### Context Provider

React 上下文可能是一个很难使用的工具——如果你传入的值变化太频繁，它可能会导致整个树的重新渲染。 这意味着我们需要传递尽可能少变化的值。

幸运的是，XState 为我们提供了一个一流的方法：`useInterpret`。

```js
import React, { createContext } from 'react';
import { useInterpret } from '@xstate/react';
import { authMachine } from './authMachine';

export const GlobalStateContext = createContext({});

export const GlobalStateProvider = (props) => {
  const authService = useInterpret(authMachine);

  return (
    <GlobalStateContext.Provider value={{ authService }}>
      {props.children}
    </GlobalStateContext.Provider>
  );
};
```

使用 `useInterpret` 返回一个服务，它是对可以订阅的正在运行的机器的静态引用。 这个值永远不会改变，所以我们不需要担心浪费的重新渲染。

> 对于 Typescript，您可以将上下文创建为 `createContext({} as InterpreterFrom<typeof authMachine>);` 以确保强类型化。

### 利用 context

在子级，您可以像这样订阅服务：

```js
import React, { useContext } from 'react';
import { GlobalStateContext } from './globalState';
import { useActor } from '@xstate/react';

export const SomeComponent = (props) => {
  const globalServices = useContext(GlobalStateContext);
  const [state] = useActor(globalServices.authService);

  return state.matches('loggedIn') ? 'Logged In' : 'Logged Out';
};
```

`useActor` 会监听服务何时更改，并更新状态值。

### 提升性能

上面的实现存在问题 - 这将更新组件以对服务进行任何更改。 [Redux](https://redux.js.org) 之类的工具使用 [`selectors`](https://redux.js.org/usage/deriving-data-selectors) 来获取状态。 选择器是限制状态的哪些部分可能导致组件重新渲染的功能。

幸运的是，XState 公开了 `useSelector` hook。

```js
import React, { useContext } from 'react';
import { GlobalStateContext } from './globalState';
import { useSelector } from '@xstate/react';

const loggedInSelector = (state) => {
  return state.matches('loggedIn');
};

export const SomeComponent = (props) => {
  const globalServices = useContext(GlobalStateContext);
  const isLoggedIn = useSelector(globalServices.authService, loggedInSelector);

  return isLoggedIn ? 'Logged In' : 'Logged Out';
};
```

如果需要在消费服务的组件中发送事件，可以直接使用`service.send(...)`方法：

```js
import React, { useContext } from 'react';
import { GlobalStateContext } from './globalState';
import { useSelector } from '@xstate/react';

const loggedInSelector = (state) => {
  return state.matches('loggedIn');
};

export const SomeComponent = (props) => {
  const globalServices = useContext(GlobalStateContext);
  const isLoggedIn = useSelector(globalServices.authService, loggedInSelector);
// 从服务中获取 `send()` 方法
  const { send } = globalServices.authService;

  return (
    <>
      {isLoggedIn && (
        <button type="button" onClick={() => send('LOG_OUT')}>
          Logout
        </button>
      )}
    </>
  );
};
```

只有当 `state.matches('loggedIn')` 返回不同的值时，此组件才会重新渲染。 当您想要优化性能时，这是我们推荐的优于 `useActor` 的方法。

### 派发事件

为了将事件调度到全局存储，你可以直接调用服务的`send`函数。

```js
import React, { useContext } from 'react';
import { GlobalStateContext } from './globalState';

export const SomeComponent = (props) => {
  const globalServices = useContext(GlobalStateContext);

  return (
    <button onClick={() => globalServices.authService.send('LOG_OUT')}>
      Log Out
    </button>
  );
};
```

请注意，你不需要为此调用 `useActor`，它可以在上下文中使用。

## 其他 hooks

XState 的 `useMachine` 和 `useInterpret` hook 可以与其他 hook 一起使用。 最常见的两种模式：

### 命名的 actions/services/guards

让我们想象一下，当您导航到某个状态时，您想通过`react-router`或`next`离开页面并转到其他地方。 现在，我们将该动作声明为“命名”动作——我们现在命名它并稍后声明它。

```js
import { createMachine } from 'xstate';

export const machine = createMachine({
  initial: 'toggledOff',
  states: {
    toggledOff: {
      on: {
        TOGGLE: 'toggledOn'
      }
    },
    toggledOn: {
      entry: ['goToOtherPage']
    }
  }
});
```

在你的组件中，你现在可以 _实现_ 指定的动作。 我已经从 `react-router` 添加了 `useHistory` 作为示例，但是你可以想象这可以与任何基于 hook 或 prop 的路由器一起使用。

```js
import { machine } from './machine';
import { useMachine } from '@xstate/react';
import { useHistory } from 'react-router';

const Component = () => {
  const history = useHistory();

  const [state, send] = useMachine(machine, {
    actions: {
      goToOtherPage: () => {
        history.push('/other-page');
      }
    }
  });

  return null;
};
```

这也适合于 services, guards, 和 delays.

> 如果你使用此技术，你在 `goToOtherPage` 中使用的任何引用都将在每次渲染时保持最新。 这意味着你无需担心过时的引用。

### 使用 useEffect 同步数据

有时，你想将某些功能外包给另一个 hook。 这在诸如 [`react-query`](https://react-query.tanstack.com/) 和 [`swr`](https://swr.vercel.app/) 之类的数据获取 hook 中尤其常见。 你不想在 XState 中重新构建所有数据获取功能。

最好的管理方法是通过`useEffect`。

```js
const Component = () => {
  const { data, error } = useSWR('/api/user', fetcher);

  const [state, send] = useMachine(machine);

  useEffect(() => {
    send({
      type: 'DATA_CHANGED',
      data,
      error
    });
  }, [data, error, send]);
};
```

每当 `useSWR` 的结果发生变化时，这将发送一个 `DATA_CHANGED` 事件，允许你像任何其他事件一样对其做出反应。 例如，你可以：

- 当数据返回错误时进入`errored`状态
- 将数据保存到上下文

## Class 组件

- 如果你使用的是类组件，这里有一个不依赖于 hooks 的示例实现。
`machine` 被 [interpreted](../guides/interpretation.md)，并且它的 `service` 实例被放置在组件实例上。
- 对于本地状态， `this.state.current` 将保存当前状态机状态。 你可以使用 `.current` 以外的属性名称。
- 当组件被挂载时，`service` 通过 `this.service.start()` 启动。
- 当组件卸载时，`service` 通过 `this.service.stop()` 停止。
- 事件通过 `this.service.send(event)` 发送到 `service`。

```jsx
import React from 'react';
import { interpret } from 'xstate';
import { toggleMachine } from '../path/to/toggleMachine';

class Toggle extends React.Component {
  state = {
    current: toggleMachine.initialState
  };

  service = interpret(toggleMachine).onTransition((current) =>
    this.setState({ current })
  );

  componentDidMount() {
    this.service.start();
  }

  componentWillUnmount() {
    this.service.stop();
  }

  render() {
    const { current } = this.state;
    const { send } = this.service;

    return (
      <button onClick={() => send('TOGGLE')}>
        {current.matches('inactive') ? 'Off' : 'On'}
      </button>
    );
  }
}
```
