---
title: 逃生舱口
---

<Intro>

你的一些组件可能需要控制并与React之外的系统同步。例如，你可能需要使用浏览器API聚焦一个输入，播放和暂停一个没有React实现的视频播放器，或者连接和收听来自远程服务器的消息。在本章中，你将学习让你 “走出” React并连接到外部系统的逃生舱口。你的大部分应用逻辑和数据流不应该依赖于这些功能。

</Intro>

<YouWillLearn isChapter={true}>

* [如何在不重新渲染的情况下 “记住” 信息](/learn/referencing-values-with-refs)
* [如何访问由React管理的DOM元素](/learn/manipulating-the-dom-with-refs)
* [如何使组件与外部系统同步](/learn/synchronizing-with-effects)
* [如何从你的组件中移除不必要的Effect](/learn/you-might-not-need-an-effect)
* [Effect的生命周期与组件的生命周期有什么不同](/learn/lifecycle-of-reactive-effects)
* [如何防止某些值重新触发Effect](/learn/separating-events-from-effects)
* [如何使你的Effect重新运行的频率降低](/learn/removing-effect-dependencies)
* [如何在组件之间共享逻辑](/learn/reusing-logic-with-custom-hooks)

</YouWillLearn>

## 用 ref 来引用数值 {/*referencing-values-with-refs*/}

当你想让一个组件 “记住” 一些信息，但又不想让这些信息[触发新的渲染](/learn/render-and-commit)时，你可以使用一个 *ref* 。

```js
const ref = useRef(0);
```

像状态一样，ref 在重新渲染时被React保留。然而，设置 state 会重新渲染一个组件，而改变一个 ref 则不会！你可以通过`ref.current`属性访问该 ref 的当前值。

<Sandpack>

```js
import { useRef } from 'react';

export default function Counter() {
  let ref = useRef(0);

  function handleClick() {
    ref.current = ref.current + 1;
    alert('You clicked ' + ref.current + ' times!');
  }

  return (
    <button onClick={handleClick}>
      Click me!
    </button>
  );
}
```

</Sandpack>

ref 就像你的组件的一个秘密口袋，React不会跟踪。例如，你可以使用 ref 来存储[超时 ID](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout#return_value)、[DOM 元素](https://developer.mozilla.org/en-US/docs/Web/API/Element)和其他不影响组件渲染输出的对象。

<LearnMore path="/learn/referencing-values-with-refs">

阅读 **[用 Ref 来引用数值](/learn/manipulating-the-dom-with-refs)** ，了解如何使用 ref 来记忆信息。

</LearnMore>

## 用 ref 操纵DOM {/*manipulating-the-dom-with-refs*/}

React会自动更新DOM以匹配你的渲染输出，所以你的组件不会经常需要操作它。然而，有时你可能需要访问由React管理的DOM元素--例如，聚焦一个节点，滚动到它，或测量它的大小和位置。在React中没有内置的方法来做这些事情，所以你将需要一个 ref 来存储DOM节点。例如，使用 ref 来实现点击按钮聚焦输入：

<Sandpack>

```js
import { useRef } from 'react';

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>
        Focus the input
      </button>
    </>
  );
}
```

</Sandpack>

<LearnMore path="/learn/manipulating-the-dom-with-refs">

阅读 **[用 Ref 操纵DOM](/learn/manipulating-the-dom-with-refs)** 来学习如何访问由React管理的DOM元素。

</LearnMore>

## 用 Effect 进行同步 {/*synchronizing-with-effects*/}

有些组件需要与外部系统同步。例如，你可能想根据React状态来控制一个非React组件，建立一个服务器连接，或者当一个组件出现在屏幕上时发送一个分析日志。与让你处理特定事件的事件处理程序不同， *Effect* 让你在渲染后运行一些代码。使用它们来使你的组件与React之外的一些系统同步。


多次按下播放/暂停，看看视频播放器如何与isPlaying `prop` 值保持同步：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  }, [isPlaying]);

  return <video ref={ref} src={src} loop playsInline />;
}

export default function App() {
  const [isPlaying, setIsPlaying] = useState(false);
  return (
    <>
      <button onClick={() => setIsPlaying(!isPlaying)}>
        {isPlaying ? 'Pause' : 'Play'}
      </button>
      <VideoPlayer
        isPlaying={isPlaying}
        src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
      />
    </>
  );
}
```

```css
button { display: block; margin-bottom: 20px; }
video { width: 250px; }
```

</Sandpack>

许多 Effect 也需要 “清理” 自己。例如，如果你的 Effect 建立了一个与聊天服务器的连接，它应该返回一个清理函数，告诉React如何将你的组件与该服务器断开连接：

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

export default function ChatRoom() {
  useEffect(() => {
    const connection = createConnection();
    connection.connect();
    return () => connection.disconnect();
  }, []);
  return <h1>Welcome to the chat!</h1>;
}
```

```js chat.js
export function createConnection() {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting...');
    },
    disconnect() {
      console.log('❌ Disconnected.');
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
```

</Sandpack>

在开发中，React会立即运行并多清理一次你的 Effect 。这就是为什么你会看到 `“✅ Connecting...”` 被打印了两次。这可以确保你不会忘记实现清理功能。

<LearnMore path="/learn/synchronizing-with-effects">

阅读 **[用 Effect 进行同步](/learn/synchronizing-with-effects)** ，了解如何使组件与外部系统同步。

</LearnMore>

## 你可能不需要 Effect {/*you-might-not-need-an-effect*/}

Effect 是React范式中的一个逃生舱口。它们让你 “跳出” React，将你的组件与一些外部系统同步。如果不涉及外部系统（例如，如果你想在某些道具或状态改变时更新组件的状态），你不应该需要 Effect 。移除不必要的 Effect 将使你的代码更容易理解，运行速度更快，并且更不容易出错。

有两种常见的情况，你不需要 Effect ：
- **你不需要 Effect 来转换数据进行渲染。**
- **你不需要 Effect 来处理用户事件。**

例如，你不需要一个 Effect 来根据其他状态来调整一些状态：

```js {5-9}
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');

  // 🔴 Avoid: redundant state and unnecessary Effect
  const [fullName, setFullName] = useState('');
  useEffect(() => {
    setFullName(firstName + ' ' + lastName);
  }, [firstName, lastName]);
  // ...
}
```

相反，在渲染时尽可能多地计算：

```js {4-5}
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');
  // ✅ Good: calculated during rendering
  const fullName = firstName + ' ' + lastName;
  // ...
}
```

然而，你*确实*需要 Effect 来与外部系统同步。

<LearnMore path="/learn/you-might-not-need-an-effect">

阅读 **[你可能不需要 Effect ](/learn/you-might-not-need-an-effect)**，了解如何删除不必要的 Effect 。

</LearnMore>

## 响应式 Effect 的生命周期 {/*lifecycle-of-reactive-effects*/}

Effect 有一个与组件不同的生命周期。组件可以装载、更新或卸载。一个 Effect 只能做两件事：开始同步某些东西，然后停止同步。如果你的 Effect 依赖于随时间变化的 prop 和 state，那么这个周期可以发生多次。

这个 Effect 取决于 `roomId` prop 的值。prop 是响应式数值，这意味着它们可以在重新渲染时发生变化。请注意，在你更新 `roomId` 之后，Effect 会重新同步（并重新连接到服务器）：

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  return <h1>Welcome to the {roomId} room!</h1>;
}

export default function App() {
  const [roomId, setRoomId] = useState('general');
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <hr />
      <ChatRoom roomId={roomId} />
    </>
  );
}
```

```js chat.js
export function createConnection(serverUrl, roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '" room at ' + serverUrl + '...');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room at ' + serverUrl);
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
button { margin-left: 10px; }
```

</Sandpack>

React提供了一个linter规则来检查你是否正确指定了你的 Effect 的依赖关系。如果你在上面的例子中忘记在依赖列表中指定 `roomId`，linter会自动找到这个错误。

<LearnMore path="/learn/lifecycle-of-reactive-effects">

阅读 **[响应式 Effect 的生命周期](/learn/lifecycle-of-reactive-effects)**，了解 Effect 的生命周期与组件的生命周期有什么不同。

</LearnMore>

## 从 Effect 中分离 Event {/*separating-events-from-effects*/}

<Wip>

本节描述了一个 **实验性的API，它还没有被添加到React中，** 所以你还不能使用它。

</Wip>

Event 只在你再次执行相同的交互时重新运行。与 Event 不同的是，如果 Effect 读取的某些值（如 prop 或 state ）与上次渲染时的值不同，它就会重新同步。有时，你想要两种行为的混合：一个对某些值重新运行的 Effect，而不是其他值。

Effect 里面的所有代码都是响应式的。如果它所读取的某些响应式数值由于重新渲染而发生了变化，它将再次运行。例如，如果在互动之后，`roomId` 或 `theme` 发生了变化，该 Effect 将重新连接到聊天：

<Sandpack>

```json package.json hidden
{
  "dependencies": {
    "react": "latest",
    "react-dom": "latest",
    "react-scripts": "latest",
    "toastify-js": "1.12.0"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --env=jsdom",
    "eject": "react-scripts eject"
  }
}
```

```js
import { useState, useEffect } from 'react';
import { createConnection, sendMessage } from './chat.js';
import { showNotification } from './notifications.js';

const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId, theme }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.on('connected', () => {
      showNotification('Connected!', theme);
    });
    connection.connect();
    return () => connection.disconnect();
  }, [roomId, theme]);

  return <h1>Welcome to the {roomId} room!</h1>
}

export default function App() {
  const [roomId, setRoomId] = useState('general');
  const [isDark, setIsDark] = useState(false);
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <label>
        <input
          type="checkbox"
          checked={isDark}
          onChange={e => setIsDark(e.target.checked)}
        />
        Use dark theme
      </label>
      <hr />
      <ChatRoom
        roomId={roomId}
        theme={isDark ? 'dark' : 'light'} 
      />
    </>
  );
}
```

```js chat.js
export function createConnection(serverUrl, roomId) {
  // A real implementation would actually connect to the server
  let connectedCallback;
  let timeout;
  return {
    connect() {
      timeout = setTimeout(() => {
        if (connectedCallback) {
          connectedCallback();
        }
      }, 100);
    },
    on(event, callback) {
      if (connectedCallback) {
        throw Error('Cannot add the handler twice.');
      }
      if (event !== 'connected') {
        throw Error('Only "connected" event is supported.');
      }
      connectedCallback = callback;
    },
    disconnect() {
      clearTimeout(timeout);
    }
  };
}
```

```js notifications.js
import Toastify from 'toastify-js';
import 'toastify-js/src/toastify.css';

export function showNotification(message, theme) {
  Toastify({
    text: message,
    duration: 2000,
    gravity: 'top',
    position: 'right',
    style: {
      background: theme === 'dark' ? 'black' : 'white',
      color: theme === 'dark' ? 'white' : 'black',
    },
  }).showToast();
}
```

```css
label { display: block; margin-top: 10px; }
```

</Sandpack>

这并不理想。您希望只有在 `roomId` 发生变化时才重新连接到聊天。切换 `theme` 不应该重新连接到聊天！将读取 `theme` 的代码从你的 Effect 中移出，放到一个*事件函数*中：

<Sandpack>

```json package.json hidden
{
  "dependencies": {
    "react": "experimental",
    "react-dom": "experimental",
    "react-scripts": "latest",
    "toastify-js": "1.12.0"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --env=jsdom",
    "eject": "react-scripts eject"
  }
}
```

```js
import { useState, useEffect } from 'react';
import { experimental_useEvent as useEvent } from 'react';
import { createConnection, sendMessage } from './chat.js';
import { showNotification } from './notifications.js';

const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId, theme }) {
  const onConnected = useEvent(() => {
    showNotification('Connected!', theme);
  });

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.on('connected', () => {
      onConnected();
    });
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  return <h1>Welcome to the {roomId} room!</h1>
}

export default function App() {
  const [roomId, setRoomId] = useState('general');
  const [isDark, setIsDark] = useState(false);
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <label>
        <input
          type="checkbox"
          checked={isDark}
          onChange={e => setIsDark(e.target.checked)}
        />
        Use dark theme
      </label>
      <hr />
      <ChatRoom
        roomId={roomId}
        theme={isDark ? 'dark' : 'light'} 
      />
    </>
  );
}
```

```js chat.js
export function createConnection(serverUrl, roomId) {
  // A real implementation would actually connect to the server
  let connectedCallback;
  let timeout;
  return {
    connect() {
      timeout = setTimeout(() => {
        if (connectedCallback) {
          connectedCallback();
        }
      }, 100);
    },
    on(event, callback) {
      if (connectedCallback) {
        throw Error('Cannot add the handler twice.');
      }
      if (event !== 'connected') {
        throw Error('Only "connected" event is supported.');
      }
      connectedCallback = callback;
    },
    disconnect() {
      clearTimeout(timeout);
    }
  };
}
```

```js notifications.js hidden
import Toastify from 'toastify-js';
import 'toastify-js/src/toastify.css';

export function showNotification(message, theme) {
  Toastify({
    text: message,
    duration: 2000,
    gravity: 'top',
    position: 'right',
    style: {
      background: theme === 'dark' ? 'black' : 'white',
      color: theme === 'dark' ? 'white' : 'black',
    },
  }).showToast();
}
```

```css
label { display: block; margin-top: 10px; }
```

</Sandpack>

事件函数内的代码不是响应式的，所以改变 `theme` 不再使你的 Effect 重新连接。

<LearnMore path="/learn/separating-events-from-effects">

阅读 **[从 Effect 中分离 Event](/learn/separating-events-from-effects)**，了解如何防止某些值重新触发效果。

</LearnMore>

## 删除 Effect 依赖项 {/*removing-effect-dependencies*/}

当你写一个 Effect 时，linter会验证你是否将 Effect 读取的每一个响应值（如 prop 和 state ）都包含在你的 Effect 的依赖关系列表中。这可以确保你的 Effect 与你的组件的最新 prop 和 state 保持同步。不必要的依赖关系可能会导致你的 Effect 运行过于频繁，甚至产生一个无限循环。何时删除它们取决于具体情况。

例如，这个 Effect 依赖于 `option` 对象，每次你编辑输入时都会重新创建：

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId }) {
  const [message, setMessage] = useState('');

  const options = {
    serverUrl: serverUrl,
    roomId: roomId
  };

  useEffect(() => {
    const connection = createConnection(options);
    connection.connect();
    return () => connection.disconnect();
  }, [options]);

  return (
    <>
      <h1>Welcome to the {roomId} room!</h1>
      <input value={message} onChange={e => setMessage(e.target.value)} />
    </>
  );
}

export default function App() {
  const [roomId, setRoomId] = useState('general');
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <hr />
      <ChatRoom roomId={roomId} />
    </>
  );
}
```

```js chat.js
export function createConnection({ serverUrl, roomId }) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '" room at ' + serverUrl + '...');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room at ' + serverUrl);
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
button { margin-left: 10px; }
```

</Sandpack>

你不希望每次你开始在该聊天室输入信息时，聊天室都要重新连接。为了解决这个问题，把 `option` 对象的创建移到 Effect 里面，这样 Effect 就只依赖于 `roomId` 字符串：

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId }) {
  const [message, setMessage] = useState('');

  useEffect(() => {
    const options = {
      serverUrl: serverUrl,
      roomId: roomId
    };
    const connection = createConnection(options);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  return (
    <>
      <h1>Welcome to the {roomId} room!</h1>
      <input value={message} onChange={e => setMessage(e.target.value)} />
    </>
  );
}

export default function App() {
  const [roomId, setRoomId] = useState('general');
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <hr />
      <ChatRoom roomId={roomId} />
    </>
  );
}
```

```js chat.js
export function createConnection({ serverUrl, roomId }) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '" room at ' + serverUrl + '...');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room at ' + serverUrl);
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
button { margin-left: 10px; }
```

</Sandpack>

请注意，一开始就编辑依赖关系列表来删除 `options` 这个依赖是不对的。相反，你应该首先改变周围的代码，使这个依赖关系变得*不必要*，这时候再删除`options` 这个依赖。你可以把依赖关系列表看作是你的 Effect 代码所使用的所有响应值的列表，你不会有意选择把什么放在这个列表上。这个列表描述了你的代码，要改变依赖关系列表，就要改变代码。

<LearnMore path="/learn/removing-effect-dependencies">

阅读 **[删除 Effect 依赖项](/learn/removing-effect-dependencies)** ，了解如何使你的 Effect 更少地重新运行。

</LearnMore>

## 用自定义 Hook 复用逻辑 {/*reusing-logic-with-custom-hooks*/}

React有内置的 Hook，比如 `useState`、`useContext` 和 `useEffect`。有时，你会希望有一个 Hook 用于某些更具体的目的：例如，获取数据，跟踪用户是否在线，或连接到一个聊天室。要做到这一点，你可以为你的应用程序的需要创建你自己的 Hook。

在这个例子中，`usePointerPosition` 自定义钩子跟踪光标的位置，而 `useDelayedValue` 自定义钩子返回一个“滞后”值，滞后时间取决于你传的毫秒数。将光标移到沙盒预览区，可以看到光标后面有一串移动的小点。

<Sandpack>

```js
import { usePointerPosition } from './usePointerPosition.js';
import { useDelayedValue } from './useDelayedValue.js';

export default function Canvas() {
  const pos1 = usePointerPosition();
  const pos2 = useDelayedValue(pos1, 100);
  const pos3 = useDelayedValue(pos2, 200);
  const pos4 = useDelayedValue(pos3, 100);
  const pos5 = useDelayedValue(pos3, 50);
  return (
    <>
      <Dot position={pos1} opacity={1} />
      <Dot position={pos2} opacity={0.8} />
      <Dot position={pos3} opacity={0.6} />
      <Dot position={pos4} opacity={0.4} />
      <Dot position={pos5} opacity={0.2} />
    </>
  );
}

function Dot({ position, opacity }) {
  return (
    <div style={{
      position: 'absolute',
      backgroundColor: 'pink',
      borderRadius: '50%',
      opacity,
      transform: `translate(${position.x}px, ${position.y}px)`,
      pointerEvents: 'none',
      left: -20,
      top: -20,
      width: 40,
      height: 40,
    }} />
  );
}
```

```js usePointerPosition.js
import { useState, useEffect } from 'react';

export function usePointerPosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  useEffect(() => {
    function handleMove(e) {
      setPosition({ x: e.clientX, y: e.clientY });
    }
    window.addEventListener('pointermove', handleMove);
    return () => window.removeEventListener('pointermove', handleMove);
  }, []);
  return position;
}
```

```js useDelayedValue.js
import { useState, useEffect } from 'react';

export function useDelayedValue(value, delay) {
  const [delayedValue, setDelayedValue] = useState(value);

  useEffect(() => {
    setTimeout(() => {
      setDelayedValue(value);
    }, delay);
  }, [value, delay]);

  return delayedValue;
}
```

```css
body { min-height: 300px; }
```

</Sandpack>

你可以创建自定义 Hook，将它们组合在一起，在它们之间传递数据，并在组件之间重复使用它们。随着你的应用程序的增长，你将会减少手工编写的 Effect，因为你能够重复使用你已经编写的自定义 Hook。还有许多优秀的自定义 Hook 由React社区维护。

<LearnMore path="/learn/reusing-logic-with-custom-hooks">

阅读 **[用自定义 Hook 复用逻辑](/learn/reusing-logic-with-custom-hooks)**，了解如何在组件之间共享逻辑。

</LearnMore>

## 下一步是什么？ {/*whats-next*/}

请到[用 ref 来引用数值](/learn/referencing-values-with-refs)栏目，开始逐页阅读本章内容！
