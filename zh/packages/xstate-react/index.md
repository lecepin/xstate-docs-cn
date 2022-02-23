# @xstate/react

[@xstate/react 包](https://github.com/statelyai/xstate/tree/main/packages/xstate-react) 包含使用 [XState](https://github.com/statelyai/xstate) 与 [React](https://github.com/facebook/react/) 的实用程序。

[[toc]]

## 快速开始

1. 安装 `xstate` 和 `@xstate/react`:

```bash
npm i xstate @xstate/react
```

**通过 CDN**

```html
<script src="https://unpkg.com/@xstate/react/dist/xstate-react.umd.min.js"></script>
```

通过使用全局变量 `XStateReact`

或

```html
<script src="https://unpkg.com/@xstate/react/dist/xstate-react-fsm.umd.min.js"></script>
```

通过使用全局变量 `XStateReactFSM`

2. 导入 `useMachine` hook:

```js
import { useMachine } from '@xstate/react';
import { createMachine } from 'xstate';

const toggleMachine = createMachine({
  id: 'toggle',
  initial: 'inactive',
  states: {
    inactive: {
      on: { TOGGLE: 'active' }
    },
    active: {
      on: { TOGGLE: 'inactive' }
    }
  }
});

export const Toggler = () => {
  const [state, send] = useMachine(toggleMachine);

  return (
    <button onClick={() => send('TOGGLE')}>
      {state.value === 'inactive'
        ? 'Click to activate'
        : 'Active! Click to deactivate'}
    </button>
  );
};
```

## 示例

- [XState + React TodoMVC (CodeSandbox)](https://codesandbox.io/s/xstate-todomvc-33wr94qv1)

## API

### `useMachine(machine, options?)`

一个 [React hook](https://reactjs.org/hooks)，它解释给定的 `machine` 并启动一个在组件的生命周期内运行的服务。

**参数**

- `machine` - [XState machine](https://xstate.js.org/docs/guides/machines.html) 或延迟返回 machine 的函数：

  ```js
  // 现在的 machine
  const [state, send] = useMachine(machine);

  // 延迟创建的 machine
  const [state, send] = useMachine(() =>
    createMachine({
      /* ... */
    })
  );
  ```

- `options` (optional) - [Interpreter options](https://xstate.js.org/docs/guides/interpretation.html#options) and/or 以下任何 machine 配置选项: `guards`, `actions`, `services`, `delays`, `immediate`, `context`, `state`.

**返回值** 一个元组 `[state, send, service]`:

- `state` - 将 machine 的当前状态表示为 XState `State` 对象。
- `send` - 向正在运行的服务发送事件的函数。
- `service` - 创建的服务。

### `useService(service)`

::: warning Deprecated

在下一个主要版本中，`useService(service)` 将被替换为 `useActor(service)`。 更喜欢使用 `useActor(service)` hook 来代替服务，因为服务也是演员。

另外，请记住，只有一个参数（事件对象）可以从 `useActor(...)` 发送到 `send(eventObject)`。 迁移到 `useActor(...)` 时，重构 `send(...)` 调用以仅使用单个事件对象：

```diff
const [state, send] = useActor(service);
-send('CLICK', { x: 0, y: 3 });
+send({ type: 'CLICK', x: 0, y: 3 });
```

:::

订阅现有 [服务] (https://xstate.js.org/docs/guides/interpretation.html) 的状态更改的 [React hook](https://reactjs.org/hooks)。

**参数**

- `service` - [XState 服务](https://xstate.js.org/docs/guides/interpretation.html)。

**返回值** a tuple of `[state, send]`:

- `state` - 将服务的当前状态表示为 XState `State` 对象。
- `send` - 向正在运行的服务发送事件的函数。

### `useActor(actor, getSnapshot?)`

订阅现有 [actor](https://xstate.js.org/docs/guides/actors.html) 发出更改的 [React hook](https://reactjs.org/hooks)。

**参数**

- `actor` - 一个类似actor的对象，包含`.send(...)`和`.subscribe(...)`方法。
- `getSnapshot` - 一个应该从 `actor` 返回最新发出的值的函数。
  - 默认尝试获取 `actor.state`，如果不存在则返回 `undefined`。

```js
const [state, send] = useActor(someSpawnedActor);

// 自定义演员
const [state, send] = useActor(customActor, (actor) => {
  // 特定于实现的伪代码示例：
  return actor.getLastEmittedValue();
});
```

### `useInterpret(machine, options?, observer?)`

一个 React hook，它返回从带有 `options` 的 `machine` 创建的 `service`，如果指定的话。 它启动服务并在组件的生命周期内运行它。 这类似于`useMachine`； 但是，`useInterpret` 允许自定义的`observer` 订阅`service`。

当你需要细粒度控制时，`useInterpret` 很有用，例如 添加日志记录，或最小化重新渲染。 与将每次更新从机器刷新到 React 组件的 `useMachine` 相比，`useInterpret` 返回一个静态引用（仅对解释的 machine），当其状态改变时不会重新渲染。

要在渲染中使用服务中的某个状态，请使用 `useSelector(...)` hook 来订阅它。

_Since 1.3.0_

**参数**

- `machine` - [XState machine](https://xstate.js.org/docs/guides/machines.html) 或延迟返回 machine 的函数。
- `options` (optional) - [Interpreter options](https://xstate.js.org/docs/guides/interpretation.html#options) and/or 以下任何 machine 配置选项: `guards`, `actions`, `services`, `delays`, `immediate`, `context`, `state`.
- `observer` (optional) - 监听状态更新的观察者或监听者：
  - an observer (e.g., `{ next: (state) => {/* ... */} }`)
  - or a listener (e.g., `(state) => {/* ... */}`)

```js
import { useInterpret } from '@xstate/react';
import { someMachine } from '../path/to/someMachine';

const App = () => {
  const service = useInterpret(someMachine);

  // ...
};
```

With options + listener:

```js
// ...

const App = () => {
  const service = useInterpret(
    someMachine,
    {
      actions: {
        /* ... */
      }
    },
    (state) => {
      // 订阅状态更改
      console.log(state);
    }
  );

  // ...
};
```

### `useSelector(actor, selector, compare?, getSnapshot?)`

一个 React hook，它从 `actor` 的快照中返回选定的值，例如服务。 如果选择的值发生变化，这个 hook 只会导致重新渲染，由可选的 `compare` 函数确定。

_Since 1.3.0_

**参数**

- `actor` - 包含 `.send(...)` 和 `.subscribe(...)` 方法的服务或类似actor的对象。
- `selector` - 一个函数，它将参与者的“当前状态”（快照）作为参数并返回所需的选定值。
- `compare` (optional) - 确定当前选定值是否与先前选定值相同的函数。
- `getSnapshot` (optional) - 一个应该从 `actor` 返回最新发出的值的函数。
  - 默认尝试获取 `actor.state`，如果不存在则返回 `undefined`。 将自动从服务中提取状态。

```js
import { useSelector } from '@xstate/react';

// 提示：尽可能通过在外部定义选择器来优化选择器
const selectCount = (state) => state.context.count;

const App = ({ service }) => {
  const count = useSelector(service, selectCount);

  // ...
};
```

With `compare` function:

```js
// ...

const selectUser = (state) => state.context.user;
const compareUser = (prevUser, nextUser) => prevUser.id === nextUser.id;

const App = ({ service }) => {
  const user = useSelector(service, selectUser, compareUser);

  // ...
};
```

With `useInterpret(...)`:

```js
import { useInterpret, useSelector } from '@xstate/react';
import { someMachine } from '../path/to/someMachine';

const selectCount = (state) => state.context.count;

const App = ({ service }) => {
  const service = useInterpret(someMachine);
  const count = useSelector(service, selectCount);

  // ...
};
```

### `asEffect(action)`

确保 `action` 在 `useEffect` 中作为副作用执行，而不是立即执行。

**参数**

- `action` - 一个 action 函数 (e.g., `(context, event) => { alert(context.message) })`)

**返回值** 一个特殊的 action 函数，它封装了原始函数，以便 `useMachine` 知道在 `useEffect` 中执行它。

**示例**

```jsx
const machine = createMachine({
  initial: 'focused',
  states: {
    focused: {
      entry: 'focus'
    }
  }
});

const Input = () => {
  const inputRef = useRef(null);
  const [state, send] = useMachine(machine, {
    actions: {
      focus: asEffect((context, event) => {
        inputRef.current && inputRef.current.focus();
      })
    }
  });

  return <input ref={inputRef} />;
};
```

### `asLayoutEffect(action)`

确保 `action` 在 `useEffect` 中作为副作用执行，而不是立即执行。

**参数**

- `action` - 一个 action 函数 (e.g., `(context, event) => { alert(context.message) })`)

**返回值** 一个特殊的 action 函数，它封装了原始函数，以便 `useMachine` 知道在 `useEffect` 中执行它。

### `useMachine(machine)` with `@xstate/fsm`

一个 [React hook](https://reactjs.org/hooks) 从 [`@xstate/fsm`] 解释给定的有限状态 `machine` 并启动一个在组件的生命周期内运行的服务。

这个特殊的 `useMachine` 钩子是从 `@xstate/react/fsm` 导入的

**参数**

- `machine` - [XState 有限状态机 (FSM)](https://xstate.js.org/docs/packages/xstate-fsm/)。
- `options` - 一个可选的 `options` 对象。

**返回值** 一个元组 `[state, send, service]`:

- `state` - 将 machine 的当前状态表示为 `@xstate/fsm` `StateMachine.State` 对象。
- `send` - 向正在运行的服务发送事件的函数。
- `service` - 创建的 `@xstate/fsm` 服务。

**示例**

```js
import { useEffect } from 'react';
import { useMachine } from '@xstate/react/fsm';
import { createMachine } from '@xstate/fsm';

const context = {
  data: undefined
};
const fetchMachine = createMachine({
  id: 'fetch',
  initial: 'idle',
  context,
  states: {
    idle: {
      on: { FETCH: 'loading' }
    },
    loading: {
      entry: ['load'],
      on: {
        RESOLVE: {
          target: 'success',
          actions: assign({
            data: (context, event) => event.data
          })
        }
      }
    },
    success: {}
  }
});

const Fetcher = ({
  onFetch = () => new Promise((res) => res('some data'))
}) => {
  const [state, send] = useMachine(fetchMachine, {
    actions: {
      load: () => {
        onFetch().then((res) => {
          send({ type: 'RESOLVE', data: res });
        });
      }
    }
  });

  switch (state.value) {
    case 'idle':
      return <button onClick={(_) => send('FETCH')}>Fetch</button>;
    case 'loading':
      return <div>Loading...</div>;
    case 'success':
      return (
        <div>
          Success! Data: <div data-testid="data">{state.context.data}</div>
        </div>
      );
    default:
      return null;
  }
};
```

## 配置 Machines

可以通过将 machines 选项作为“useMachine(machine, options)”的第二个参数传递来配置现有 machines。

示例：“fetchData”服务和“notifySuccess”操作都是可配置的：

```js
const fetchMachine = createMachine({
  id: 'fetch',
  initial: 'idle',
  context: {
    data: undefined,
    error: undefined
  },
  states: {
    idle: {
      on: { FETCH: 'loading' }
    },
    loading: {
      invoke: {
        src: 'fetchData',
        onDone: {
          target: 'success',
          actions: assign({
            data: (_, event) => event.data
          })
        },
        onError: {
          target: 'failure',
          actions: assign({
            error: (_, event) => event.data
          })
        }
      }
    },
    success: {
      entry: 'notifySuccess',
      type: 'final'
    },
    failure: {
      on: {
        RETRY: 'loading'
      }
    }
  }
});

const Fetcher = ({ onResolve }) => {
  const [state, send] = useMachine(fetchMachine, {
    actions: {
      notifySuccess: (ctx) => onResolve(ctx.data)
    },
    services: {
      fetchData: (_, e) =>
        fetch(`some/api/${e.query}`).then((res) => res.json())
    }
  });

  switch (state.value) {
    case 'idle':
      return (
        <button onClick={() => send('FETCH', { query: 'something' })}>
          Search for something
        </button>
      );
    case 'loading':
      return <div>Searching...</div>;
    case 'success':
      return <div>Success! Data: {state.context.data}</div>;
    case 'failure':
      return (
        <>
          <p>{state.context.error.message}</p>
          <button onClick={() => send('RETRY')}>Retry</button>
        </>
      );
    default:
      return null;
  }
};
```

## Matching 状态

使用 [hierarchical](https://xstate.js.org/docs/guides/hierarchical.html) 和 [parallel](https://xstate.js.org/docs/guides/parallel.html) machine 时， 状态值将是对象，而不是字符串。 在这种情况下，最好使用 [`state.matches(...)`](https://xstate.js.org/docs/guides/states.html#state-methods-and-getters)。

我们可以使用 `if/else if/else` 块来做到这一点：

```js
// ...
if (state.matches('idle')) {
  return /* ... */;
} else if (state.matches({ loading: 'user' })) {
  return /* ... */;
} else if (state.matches({ loading: 'friends' })) {
  return /* ... */;
} else {
  return null;
}
```

我们也可以继续使用`switch`，但我们必须对我们的方法进行调整。 通过将`switch`的表达式设置为`true`，我们可以使用[`state.matches(...)`](https://xstate.js.org/docs/guides/states.html#state- 方法和获取machine）作为每个`case`中的谓词：

```js
switch (true) {
  case state.matches('idle'):
    return /* ... */;
  case state.matches({ loading: 'user' }):
    return /* ... */;
  case state.matches({ loading: 'friends' }):
    return /* ... */;
  default:
    return null;
}
```

也可以考虑使用三元语句，尤其是在呈现的 JSX 中：

```jsx
const Loader = () => {
  const [state, send] = useMachine(/* ... */);

  return (
    <div>
      {state.matches('idle') ? (
        <Loader.Idle />
      ) : state.matches({ loading: 'user' }) ? (
        <Loader.LoadingUser />
      ) : state.matches({ loading: 'friends' }) ? (
        <Loader.LoadingFriends />
      ) : null}
    </div>
  );
};
```

## 持续和再融合状态

你可以通过 `options.state` 使用 `useMachine(...)` 来保持和补充状态：

```js
// ...

// 从某处获取持久状态配置对象，例如 localStorage
const persistedState = JSON.parse(localStorage.getItem('some-persisted-state-key')) || someMachine.initialState;

const App = () => {
  const [state, send] = useMachine(someMachine, {
    state: persistedState // 在此处提供持久状态配置对象
  });

  // 状态最初将是持久状态，而不是 machine 的初始状态

  return (/* ... */)
}
```

## 服务

`useMachine(machine)`中创建的`service`可以作为第三个返回值引用：

```js
//                  vvvvvvv
const [state, send, service] = useMachine(someMachine);
```

你可以使用 [`useEffect` hook](https://reactjs.org/docs/hooks-effect.html) 订阅该服务的状态更改：

```js
// ...

useEffect(() => {
  const subscription = service.subscribe((state) => {
    // 简单的状态 log
    console.log(state);
  });

  return subscription.unsubscribe;
}, [service]); // 注意：服务不应该改变
```

## 从 0.x 迁移

- 对于使用 `invoke` 或 `spawn(...)` 创建的衍生演员，请使用 `useActor()` hook 而不是 `useService()`：

  ```diff
  -import { useService } from '@xstate/react';
  +import { useActor } from '@xstate/react';

  -const [state, send] = useService(someActor);
  +const [state, send] = useActor(someActor);
  ```

## Resources

[State Machines in React](https://gedd.ski/post/state-machines-in-react/)
