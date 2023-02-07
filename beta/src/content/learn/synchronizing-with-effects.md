---
title: '用 Effect 进行同步'
---

<Intro>

有些组件需要与外部系统同步。例如，你可能想根据React状态来控制一个非React组件，建立一个服务器连接，或者当一个组件出现在屏幕上时发送一个分析日志。*Effect* 让你在渲染后运行一些代码，这样你就可以将你的组件与React之外的一些系统同步。

</Intro>

<YouWillLearn>

- 什么是 Effect
- Effect 与 Event 有什么不同
- 如何在你的组件中声明一个 Effect
- 如何避免不必要地重新运行一个 Effect
- 为什么 Effect 在开发中会运行两次以及如何解决这个问题？

</YouWillLearn>

## 什么是 Effect，它们与 Event 有什么不同？ {/*what-are-effects-and-how-are-they-different-from-events*/}

在讨论 Effect 之前，你需要熟悉React组件内部的两类逻辑：

- **渲染代码**（在[描述用户界面](/learn/describing-the-ui)中介绍）位于组件的顶层。这是你获取 prop 和 state，转换它们，并返回你想在屏幕上看到的JSX的地方。渲染代码必须是纯粹的。就像一个数学公式，它应该只计算结果，而不做其他事情。

- **事件处理程序**（在[添加交互性](/learn/adding-interactivity)中介绍）是嵌套在你的组件中的函数，它可以做一些事情，而不仅仅是计算它们。一个事件处理程序可能会更新一个输入字段，提交一个 HTTP POST 请求来购买产品，或者将用户导航到另一个屏幕。事件处理程序包含 “副作用”（它们改变了程序的状态），并且是由特定的用户行为（例如，点击按钮或打字）引起。

有时这还不够。考虑一个 `ChatRoom` 组件，只要它在屏幕上可见，就必须连接到聊天服务器。连接到服务器不是一个纯粹的计算（它是一个副作用），所以它不可能在渲染过程中发生。然而，并没有像点击那样的单一特定事件导致 `ChatRoom` 被显示。

***Effect*可以让你指定由渲染本身引起的副作用，而不是由某个特定事件引起的。** 在聊天中发送消息是一个*事件*，因为它是由用户点击一个特定按钮直接引起的。然而，设置一个服务器连接是一个*Effect*，因为它需要发生，不管是哪种交互导致组件出现。Effect 在屏幕更新后的[渲染过程](/learn/render-and-commit)结束时运行（相当于是 onMounted 生命周期）。这是一个将React组件与一些外部系统（如网络或第三方库）同步的好时机。

<Note>

在这里和后面的文字中，大写的 “Effect” 指的是上面React的特定定义，即由渲染引起的副作用。为了指代更广泛的编程概念，我们会说 “副作用”。

</Note>


## 你可能不需要 Effect {/*you-might-not-need-an-effect*/}

**不要急于为你的组件添加 Effect。** 请记住，Effect 通常用于 “走出” 你的React代码并与一些外部系统同步。这包括浏览器API、第三方小工具、网络等等。如果你的 Effect 只是根据其他状态来调整一些状态，[你可能不需要Effect。](/learn/you-might-not-need-an-effect)

## 如何编写 Effect {/*how-to-write-an-effect*/}

要写一个 Effect，请遵循以下三个步骤：

1. **声明一个 Effect。** 默认情况下，你的效果将在每次渲染后运行。
2. **指定 Effect 的依赖项。** 大多数 Effetc 应该只在需要的时候重新运行，而不是在每次渲染之后。例如，淡入动画应该只在一个组件出现时触发。聊天室的连接和断开应该只在组件出现和消失时发生，或者在聊天室改变时发生。你将学习如何通过指定*依赖项*来控制这一点。
3. **如果需要的话，添加清理功能。** 一些 Effect 需要指定如何停止、撤销或清理它们正在做的事情。例如，“connect” 需要 “disconnect”，“subscribe” 需要 “unsubscribe”，而 “fetch” 需要 “cancel” 或  “ignore”。你将学习如何通过返回一个 *清理函数* 来做到这一点。

让我们详细看看这些步骤中的每一步。

### Step 1: 声明一个 Effect {/*step-1-declare-an-effect*/}

要在你的组件中声明一个 Effect，从React导入[`useEffect` Hook](/reference/react/useEffect)：

```js
import { useEffect } from 'react';
```

然后，在你的组件的顶层调用它，并在你的 Effect 里面放一些代码：

```js {2-4}
function MyComponent() {
  useEffect(() => {
    // Code here will run after *every* render
  });
  return <div />;
}
```

每次你的组件渲染时，React会更新屏幕，*然后*运行 `useEffect` 里面的代码。换句话说，**`useEffect` “推迟” 了一段代码的运行，直到渲染反映在屏幕上。**

让我们看看如何使用一个 Effect 来与外部系统同步。考虑一个 `<VideoPlayer>` React组件。如果能通过传递一个 `isPlaying` prop 来控制它是在播放还是暂停，那就更好了：

```js
<VideoPlayer isPlaying={isPlaying} />;
```

你的自定义 `VideoPlayer` 组件渲染了内置的浏览器[`<video>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/video)标签：

```js
function VideoPlayer({ src, isPlaying }) {
  // TODO: do something with isPlaying
  return <video src={src} />;
}
```

然而，浏览器的 `<video>` 标签没有 `isPlaying` prop。控制它的唯一方法是手动调用DOM元素上的[`play()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/play)和[`pause()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/pause)方法。**你需要将  `isPlaying` prop 的值与 `play()` 和 `pause()` 等方法调用同步，`isPlaying` prop 告诉你视频当前是否 _应该_ 被播放。**

我们首先需要得到一个指向`<video>` [DOM节点的 ref](/learn/manipulating-the-dom-with-refs)。

你可能会试图在渲染过程中调用 `play()` 或 `pause()`，但这并不正确：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  if (isPlaying) {
    ref.current.play();  // Calling these while rendering isn't allowed.
  } else {
    ref.current.pause(); // Also, this crashes.
  }

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

这段代码不正确的原因是，它试图在渲染过程中对DOM节点做一些事情。在React中，[渲染应该是JSX的纯计算](/learn/keeping-components-pure)，不应该包含修改DOM等副作用。

此外，当 `VideoPlayer` 第一次被调用时，它的 DOM 还不存在！还没有一个 DOM 节点来调用 `play()` 或 `pause()` ，因为React不知道要创建什么 DOM，直到你返回 JSX 之后。

这里的解决方案是**用 `useEffect` 来包裹副作用，把它从渲染计算中移出来：**

```js {6,12}
import { useEffect, useRef } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  });

  return <video ref={ref} src={src} loop playsInline />;
}
```

通过将DOM更新包裹在一个 Effect 中，你可以让React先更新屏幕。然后再运行你的Effect。

当你的 `VideoPlayer` 组件渲染时（无论是第一次还是重新渲染），有几件事会发生。首先，React会更新屏幕，确保 `<video>` 标签在DOM中具有正确的 prop。然后React会运行你的 Effect。最后，你的 Effect 将根据 `isPlaying` prop 的值调用 `play()` 或 `pause()` 。

多次按下播放/暂停，看看视频播放器如何与 `isPlaying` 值保持同步：

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
  });

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

在这个例子中，你同步到React状态的 “外部系统” 是浏览器媒体API。你可以用类似的方法将传统的非React代码（如jQuery插件）包装成声明性的React组件。

请注意，控制一个视频播放器在实践中要复杂得多。调用 `play()` 可能会失败，用户可能会使用内置的浏览器控件播放或暂停，等等。这个例子是非常简化和不完整的。

<Pitfall>

默认情况下，Effect 会在每次渲染后运行。这就是为什么像这样的代码会**产生一个无限循环：**

```js
const [count, setCount] = useState(0);
useEffect(() => {
  setCount(count + 1);
});
```

Effect 的运行是渲染的*结果*。设置状态会 *触发* 渲染。在一个 Effect 中立即设置状态，就像把电源插座插到自己身上。Effect 运行时，它设置了状态，这将导致重新渲染，这将导致 Effect 运行，它再次设置状态，这将导致另一次重新渲染，如此循环。

Effect 通常应该使你的组件与 *外部系统* 同步。如果没有外部系统，而你只想根据其他状态来调整某些状态，[你可能不需要 Effect](/learn/you-might-not-need-an-effect)。

</Pitfall>

### Step 2: 指定 Effect 依赖 {/*step-2-specify-the-effect-dependencies*/}

默认情况下，Effect 会在 *每次* 渲染后运行。通常，这**并不是你想要的**：

- 有时，它很慢。与外部系统同步并不总是即时的，所以除非有必要，否则您可能想跳过这样做。例如，您不希望每次按键都重新连接到聊天服务器。
- 有时，这是不对的。例如，你不希望在每个按键上都触发一个组件的淡入动画。该动画应该只在组件第一次出现时播放一次。

为了证明这个问题，这里是以前的例子，有几个 `console.log` 调用和一个更新父组件状态的文本输入。注意打字是如何导致 Effect 重新运行的：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      console.log('Calling video.play()');
      ref.current.play();
    } else {
      console.log('Calling video.pause()');
      ref.current.pause();
    }
  });

  return <video ref={ref} src={src} loop playsInline />;
}

export default function App() {
  const [isPlaying, setIsPlaying] = useState(false);
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
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
input, button { display: block; margin-bottom: 20px; }
video { width: 250px; }
```

</Sandpack>

你可以告诉React**跳过不必要的重新运行 Effect**，方法是在 `useEffect` 调用中指定一个 *依赖数组* 作为第二个参数。首先，在上面的例子中第 14 行添加一个空的 `[]` 数组：

```js {3}
  useEffect(() => {
    // ...
  }, []);
```

你应该看到一个错误，说 `React Hook useEffect has a missing dependency: 'isPlaying'`：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      console.log('Calling video.play()');
      ref.current.play();
    } else {
      console.log('Calling video.pause()');
      ref.current.pause();
    }
  }, []); // This causes an error

  return <video ref={ref} src={src} loop playsInline />;
}

export default function App() {
  const [isPlaying, setIsPlaying] = useState(false);
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
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
input, button { display: block; margin-bottom: 20px; }
video { width: 250px; }
```

</Sandpack>

问题是，你的 Effect 里面的代码 *依赖于* `isPlaying` prop 来决定做什么，但这个依赖关系没有被明确声明。为了解决这个问题，把 `isPlaying` 添加到依赖关系数组中：

```js {2,7}
  useEffect(() => {
    if (isPlaying) { // It's used here...
      // ...
    } else {
      // ...
    }
  }, [isPlaying]); // ...so it must be declared here!
```

现在所有的依赖关系都被声明了，所以不会有错误。指定 `[isPlaying]` 作为依赖数组告诉React，如果 `isPlaying` 与上次渲染时相同，它应该跳过重新运行你的 Effect。有了这个变化，在输入框中打字不会导致 Effect 的重新运行，但按下播放/暂停就会：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      console.log('Calling video.play()');
      ref.current.play();
    } else {
      console.log('Calling video.pause()');
      ref.current.pause();
    }
  }, [isPlaying]);

  return <video ref={ref} src={src} loop playsInline />;
}

export default function App() {
  const [isPlaying, setIsPlaying] = useState(false);
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
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
input, button { display: block; margin-bottom: 20px; }
video { width: 250px; }
```

</Sandpack>

依赖关系数组可以包含多个依赖关系。只有当你指定的所有依赖项的值与之前渲染时的值完全相同时，React才会跳过重新运行 Effect。React使用[`Object.is`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is)比较法来比较依赖关系的值。更多细节见[`useEffect` API reference](/reference/react/useEffect#reference)。

**注意，你不能 “选择” 你的依赖性。** 如果你指定的依赖关系与React根据你的 Effect 里面的代码所期望的不一致，你会得到一个 lint 错误。这有助于捕捉你代码中的许多错误。如果你的 Effect 使用了某些值，但你 *不想* 在它改变时重新运行Effect，你需要 [*编辑 Effect 代码本身*，使其不 “需要” 该依赖。](/learn/lifecycle-of-reactive-effects#what-to-do-when-you-dont-want-to-re-synchronize)

<Pitfall>

*没有* 依赖性数组和有空 `[]` 依赖性数组的行为是非常不同的：

```js {3,7,11}
useEffect(() => {
  // This runs after every render
});

useEffect(() => {
  // This runs only on mount (when the component appears)
}, []);

useEffect(() => {
  // This runs on mount *and also* if either a or b have changed since the last render
}, [a, b]);
```

我们将在下一步仔细研究 “mount” 的含义。

</Pitfall>

<DeepDive>

#### 为什么依赖关系数组中省略了ref？ {/*why-was-the-ref-omitted-from-the-dependency-array*/}

这个 Effect 同时使用了 `ref` 和 `isPlaying`，但只有 `isPlaying` 被声明为一个依赖关系：

```js {9}
function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);
  useEffect(() => {
    if (isPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  }, [isPlaying]);
```

这是因为 `ref` 对象有一个 *稳定的身份*：React保证 [你在每次渲染时都能从同一个 `useRef` 调用中得到同一个对象](/reference/react/useRef#returns)。它永远不会改变，所以它本身不会导致 Effect 的重新运行。因此，你是否包含它并不重要。包括它也是可以的：

```js {9}
function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);
  useEffect(() => {
    if (isPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  }, [isPlaying, ref]);
```

由 `useState` 返回的 [`set` 函数](/reference/react/useState#setstate) 也有稳定的身份，所以你经常会看到它们也从依赖关系中被省略。如果 linter 允许你省略一个依赖关系而不出错，那么这样做是安全的。

只有当 linter 能够 “看到” 对象是稳定的，省略总是稳定的依赖关系才会起作用。例如，如果 `ref` 是由父组件传递的，你就必须在依赖关系数组中指定它。然而，这很好，因为你不知道父组件是否总是传递同一个 ref，或者有条件地传递几个 ref 之一。因此，你的 Effect _将_ 取决于传递的是哪一个 ref。

</DeepDive>

### Step 3: 如果需要，添加清理功能 {/*step-3-add-cleanup-if-needed*/}

考虑一个不同的例子。你正在编写一个 `ChatRoom` 组件，当它出现时需要连接到聊天服务器。你得到了一个 `createConnection()` API，它返回一个带有 `connect()` 和 `disconnect()` 方法的对象。你如何在向用户显示时保持组件的连接？

首先写出 Effect 逻辑：

```js
useEffect(() => {
  const connection = createConnection();
  connection.connect();
});
```

每次重新渲染后连接到聊天室会很慢，所以你添加了依赖性数组：

```js {4}
useEffect(() => {
  const connection = createConnection();
  connection.connect();
}, []);
```

**Effect 里面的代码不使用任何 prop 或 state，所以你的依赖数组是 `[]`（空）。这告诉React只有在组件 “mounts” 时，即第一次出现在屏幕上时，才运行这段代码。**

让我们试着运行这段代码：

<Sandpack>

```js
import { useEffect } from 'react';
import { createConnection } from './chat.js';

export default function ChatRoom() {
  useEffect(() => {
    const connection = createConnection();
    connection.connect();
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

这个 Effect 只在 mount 上运行，所以你可能期望 `“✅ Connecting...”` 在控制台中被打印一次。**然而，如果你检查控制台，`“✅ Connecting...”` 会被打印两次。为什么会出现这种情况？**

想象一下，`ChatRoom` 组件是一个有许多不同屏幕的大型应用程序的一部分。用户在 `ChatRoom` 页面上开始他们的旅程。该组件安装并调用 `connection.connect()`。然后想象一下，用户导航到另一个屏幕--例如，到设置页面。`ChatRoom` 组件被卸载（unmounts）。最后，用户点击返回，`ChatRoom` 再次挂载（mounts）。这将建立第二个连接--但当用户在整个应用程序中导航时第一个连接从未被破坏过，这会导致连接的不断堆积。

如果没有大量的人工测试，这样的错误很容易被忽略。为了帮助你快速发现它们，在开发过程中，React会在每个组件初次挂载后立即重新挂载一次。**看到两次 `“✅ Connecting...”` 的日志可以帮助你注意到真正的问题：当组件卸载时，你的代码并没有关闭连接。**

为了解决这个问题，从你的 Effect 中返回一个清理函数：

```js {4-6}
  useEffect(() => {
    const connection = createConnection();
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []);
```

React会在 Effect 再次运行前每次调用你的清理函数，并在组件卸载（被删除）时最后一次调用。让我们看看清理函数实现后会发生什么：

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

现在你在开发中得到三个控制台日志：

1. `"✅ Connecting..."`
2. `"❌ Disconnected."`
3. `"✅ Connecting..."`

**这是开发中的正确行为。** 通过重新安装你的组件，React验证了导航离开和返回不会破坏你的代码。断开连接，然后再连接，这正是应该发生的事情！当你很好地实现了清理，运行一次 Effect 与运行它、清理它、再运行它之间应该没有用户可见的区别。因为React在开发中需要探测你的代码是否有bug，所以会有一个额外的 connect/disconnect 调用。这很正常，你不应该试图让它消失。

**在生产中，你只会看到 `“✅ Connecting...”` 被打印一次。** 重新装载组件只发生在开发中，以帮助你找到需要清理的 Effect。你可以关闭 [Strict Mode](/reference/react/StrictMode)来选择不参与开发行为，但我们建议保持它。这可以让你发现许多像上面这样的错误。

## 如何处理开发中 Effect 被运行两次？ {/*how-to-handle-the-effect-firing-twice-in-development*/}

React在开发过程中故意重新装载你的组件，以帮助你发现像上一个例子中的bug。**正确的问题不是 “如何运行一次Effect”，而是 “如何修复我的Effect，使其在重新挂载后仍能工作”。**

通常情况下，答案是实现清理功能。 清理功能应该停止或撤消 Effect 正在做的任何事情。经验法则是，用户不应该能够区分 Effect 运行一次（如在生产中）和 _setup → cleanup → setup_ 序列（正如你在开发中看到的）。

你要写的大部分 Effect 将符合以下常见模式之一。

### 控制非 React 小工具 {/*controlling-non-react-widgets*/}

有时你需要添加没有写入React的UI部件。例如，假设你正在向你的页面添加一个地图组件。它有一个 `setZoomLevel()` 方法，你想让缩放级别与React代码中的 `zoomLevel` 状态变量保持同步。你的 Effect 将类似于这样：

```js
useEffect(() => {
  const map = mapRef.current;
  map.setZoomLevel(zoomLevel);
}, [zoomLevel]);
```

注意，在这种情况下不需要清理。在开发中，React会调用 Effect 两次，但这不是问题，因为用相同的值调用 `setZoomLevel` 两次并不做任何事情。它可能会稍微慢一点，但这并不重要，因为重新挂载只是开发，不会发生在生产中。

有些API可能不允许你连续调用它们两次。例如，内置 [`<dialog>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement) 元素的 [`showModal`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal) 方法，如果你调用它两次，就会抛出错误。实现清理函数并使其关闭对话框：

```js {4}
useEffect(() => {
  const dialog = dialogRef.current;
  dialog.showModal();
  return () => dialog.close();
}, []);
```

在开发中，你的 Effect 将调用 `showModal()`，然后立即 `close()`，然后再次调用 `showModal()`。这与你在生产中看到的调用一次 `showModal()` 的用户可见行为是一样的。

### 订阅事件 {/*subscribing-to-events*/}

如果你的 Effect 订阅了什么，清理功能应该取消订阅：

```js {6}
useEffect(() => {
  function handleScroll(e) {
    console.log(e.clientX, e.clientY);
  }
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

在开发中，你的 Effect 将调用 `addEventListener()`，然后立即 `removeEventListener()`，然后用同一个处理程序再次 `addEventListener()`。所以一次只有一个活动的订阅。这与调用 `addEventListener()` 一次具有相同的用户可见的行为，正如你在生产中看到的那样。

### 触发动画 {/*triggering-animations*/}

如果你的 Effect 将一些东西做成了动画，清理函数应该将动画重置为初始值：

```js {4-6}
useEffect(() => {
  const node = ref.current;
  node.style.opacity = 1; // Trigger the animation
  return () => {
    node.style.opacity = 0; // Reset to the initial value
  };
}, []);
```
在开发中，不透明度将被设置为 `1`，然后设置为 `0`，然后再设置为 `1`。这应该与直接设置为 `1` 的用户可见行为相同，这就是生产中会发生的情况。如果你使用一个支持 Tweening 的第三方动画库，你的清理函数应该将 Tween 的时间线重置为初始状态。

### 请求数据 {/*fetching-data*/}

如果你的 Effect 请求了一些东西，清理函数应该[中止请求](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)或忽略其结果：

```js {2,6,13-15}
useEffect(() => {
  let ignore = false;

  async function startFetching() {
    const json = await fetchTodos(userId);
    if (!ignore) {
      setTodos(json);
    }
  }

  startFetching();

  return () => {
    ignore = true;
  };
}, [userId]);
```

你不能 “撤消” 已经发生的网络请求，但你的清理函数应该确保 _不再相关_ 的请求不会继续影响你的应用程序。例如，如果 `userId` 从 `'Alice'` 变成了 `'Bob'`，清理可以确保 `'Alice'` 的响应被忽略，即使它在 `'Bob'` 之后到达。

**在开发中，你会在网络标签中看到两个请求。** 这并没有什么不妥。用上面的方法，第一个 Effect 会立即被清理掉，所以它的 `ignore` 变量的副本会被设置为 `true`。所以即使有一个额外的请求，也不会影响状态，这要感谢 `if (!ignore)` 的检查。

**在生产中，将只有一个请求。** 如果开发中的第二个请求困扰着你，最好的办法是使用一个重复请求的解决方案，并在组件之间缓存其响应：

```js
function TodoList() {
  const todos = useSomeDataLibrary(`/api/user/${userId}/todos`);
  // ...
```

这不仅会改善开发体验，也会使你的应用程序感觉更快。例如，用户按下 “后退” 按钮就不必再等待一些数据的加载，因为这些数据会被缓存起来。你可以自己建立这样的缓存，也可以使用 Effect 中现有的许多替代手动获取的方法之一。

<DeepDive>

#### 在 Effect 中，有什么好的替代数据获取的方法？ {/*what-are-good-alternatives-to-data-fetching-in-effects*/}

在 Effect 里面编写 `fetch` 调用是一种[流行的获取数据的方式](https://www.robinwieruch.de/react-hooks-fetch-data/)，特别是在完全客户端的应用程序中。然而，这是一个非常手动的方法，它有很大的缺点：

- **Effect 不在服务器上运行。** 这意味着，最初的服务器渲染的 HTML 将只包括一个没有数据的加载状态。客户端计算机将不得不下载所有的JavaScript并渲染你的应用程序，然后发现现在它需要加载数据。这不是很有效率。
- **在 Effect 中直接获取数据使得创建 “网络瀑布” 变得容易。** 你渲染父组件，它获取一些数据，渲染子组件，然后它们开始获取它们的数据。如果网络速度不快，这比平行获取所有数据要慢得多。
- **在 Effect 中直接获取通常意味着你不需要预加载或缓存数据。** 例如，如果组件卸载了，然后再次挂载，它将不得不再次获取数据。
- **这不是很符合人体工程学。** 在 Effect 中请求数据会遇到 [race conditions](https://maxrozen.com/race-conditions-fetching-data-react-with-useeffect) 等问题，解决它们需要编写更多的模板代码。

这个缺点清单并不是专门针对React的。它适用于在挂载时获取数据的任何库。就像路由一样，数据的获取也不是那么容易做好的，所以我们推荐以下方法：

- **如果你使用一个[框架](/learn/start-a-new-react-project#building-with-a-full-featured-framework)，请使用其内置的数据获取机制。** 现代的React框架有集成的数据获取机制，效率很高，不会有上述的陷阱。
- **否则，考虑使用或建立一个客户端缓存。** 流行的开源解决方案包括[React Query](https://tanstack.com/query/latest)、[useSWR](https://swr.vercel.app/)和[React Router 6.4+](https://beta.reactrouter.com/en/main/start/overview)。你也可以建立你自己的解决方案，在这种情况下，您可以在幕后使用 Effect，还可以添加逻辑来删除重复请求、缓存响应和避免网络瀑布（通过预加载数据或将数据需求提升到路由）。

如果这两种方法都不适合你，你可以继续在 Effect 中直接获取数据。

</DeepDive>

### 发送分析报告 {/*sending-analytics*/}

考虑一下这段代码，它在页面访问时发送了一个分析事件：

```js
useEffect(() => {
  logVisit(url); // Sends a POST request
}, [url]);
```

In development, `logVisit` will be called twice for every URL, so you might be tempted to try to work around it. **We recommend to keep this code as is.** Like with earlier examples, there is no *user-visible* behavior difference between running it once and running it twice. From a practical point of view, `logVisit` should not do anything in development because you don't want the logs from the development machines to skew the production metrics. Your component remounts every time you save its file, so it would send extra visits during development anyway.

**In production, there will be no duplicate visit logs.**

To debug the analytics events you're sending, you can deploy your app to a staging environment (which runs in production mode) or temporarily opt out of [Strict Mode](/reference/react/StrictMode) and its development-only remounting checks. You may also send analytics from the route change event handlers instead of Effects. For even more precise analytics, [intersection observers](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) can help track which components are in the viewport and how long they remain visible.

### 离开 Effect：初始化应用程序 {/*not-an-effect-initializing-the-application*/}

有些逻辑应该只在应用程序启动时运行一次。你可以把它放在你的组件之外：

```js {2-3}
if (typeof window !== 'undefined') { // Check if we're running in the browser.
  checkAuthToken();
  loadDataFromLocalStorage();
}

function App() {
  // ...
}
```

这保证了这种逻辑在浏览器加载页面后只运行一次。

### 离开 Effect：购买产品 {/*not-an-effect-buying-a-product*/}

有时，即使你写了一个清理函数，也没有办法防止用户可见的运行 Effect 两次的后果。例如，也许你的 Effect 发送了一个POST请求，如购买产品：

```js {2-3}
useEffect(() => {
  // 🔴 Wrong: This Effect fires twice in development, exposing a problem in the code.
  fetch('/api/buy', { method: 'POST' });
}, []);
```

你不会想买两次产品。然而，这也是为什么你不应该把这个逻辑放在 Effect 中。如果用户去了另一个页面，然后按了 “返回” 键，怎么办？你的 Effect 就会再次运行。你不希望在用户访问一个页面时购买产品；你希望在用户 *点击* 购买按钮时购买。

购买不是由渲染引起的；它是由一个特定的互动引起的。它只运行一次，因为互动（点击）只发生一次。**删除 Effect，将你的 `/api/buy` 请求移到 Buy 按钮事件处理程序中：**

```js {2-3}
  function handleClick() {
    // ✅ Buying is an event because it is caused by a particular interaction.
    fetch('/api/buy', { method: 'POST' });
  }
```

**这说明，如果重新安装破坏了你的应用程序的逻辑，这通常会发现现有的错误。** 从用户的角度来看，访问一个页面不应该与访问它、点击一个链接、然后按 Back 不同。React通过在开发过程中重新安装一次组件来验证你的组件不会破坏这一原则。

## 把它们放在一起 {/*putting-it-all-together*/}

这个训练场可以帮助你 “感受” 到 Effect 如何在实践中发挥作用。

This example uses [`setTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout) to schedule a console log with the input text to appear three seconds after the Effect runs. The cleanup function cancels the pending timeout. Start by pressing "Mount the component":

<Sandpack>

```js
import { useState, useEffect } from 'react';

function Playground() {
  const [text, setText] = useState('a');

  useEffect(() => {
    function onTimeout() {
      console.log('⏰ ' + text);
    }

    console.log('🔵 Schedule "' + text + '" log');
    const timeoutId = setTimeout(onTimeout, 3000);

    return () => {
      console.log('🟡 Cancel "' + text + '" log');
      clearTimeout(timeoutId);
    };
  }, [text]);

  return (
    <>
      <label>
        What to log:{' '}
        <input
          value={text}
          onChange={e => setText(e.target.value)}
        />
      </label>
      <h1>{text}</h1>
    </>
  );
}

export default function App() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(!show)}>
        {show ? 'Unmount' : 'Mount'} the component
      </button>
      {show && <hr />}
      {show && <Playground />}
    </>
  );
}
```

</Sandpack>

You will see three logs at first: `Schedule "a" log`, `Cancel "a" log`, and `Schedule "a" log` again. Three second later there will also be a log saying `a`. As you learned earlier on this page, the extra schedule/cancel pair is because **React remounts the component once in development to verify that you've implemented cleanup well.**

Now edit the input to say `abc`. If you do it fast enough, you'll see `Schedule "ab" log` immediately followed by `Cancel "ab" log` and `Schedule "abc" log`. **React always cleans up the previous render's Effect before the next render's Effect.** This is why even if you type into the input fast, there is at most one timeout scheduled at a time. Edit the input a few times and watch the console to get a feel for how Effects get cleaned up.

Type something into the input and then immediately press "Unmount the component". **Notice how unmounting cleans up the last render's Effect.** In this example, it clears the last timeout before it has a chance to fire.

Finally, edit the component above and **comment out the cleanup function** so that the timeouts don't get cancelled. Try typing `abcde` fast. What do you expect to happen in three seconds? Will `console.log(text)` inside the timeout print the *latest* `text` and produce five `abcde` logs? Give it a try to check your intuition!

Three seconds later, you should see a sequence of logs (`a`, `ab`, `abc`, `abcd`, and `abcde`) rather than five `abcde` logs. **Each Effect "captures" the `text` value from its corresponding render.**  It doesn't matter that the `text` state changed: an Effect from the render with `text = 'ab'` will always see `'ab'`. In other words, Effects from each render are isolated from each other. If you're curious how this works, you can read about [closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures).

<DeepDive>

#### 每次渲染都有自己的 Effect {/*each-render-has-its-own-effects*/}

你可以把 `useEffect` 看作是把一段行为 “附加” 到渲染输出上。考虑一下这个 Effect：

```js
export default function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  return <h1>Welcome to {roomId}!</h1>;
}
```

让我们看看用户在应用中导航时到底发生了什么。

#### 初次渲染 {/*initial-render*/}

用户访问 `<ChatRoom roomId="general" />`。让我们用 `'general'` 来代替 `roomId`：

```js
  // JSX for the first render (roomId = "general")
  return <h1>Welcome to general!</h1>;
```

**Effect *也* 是渲染输出的一部分。** 第一次渲染的Effect变成了：

```js
  // Effect for the first render (roomId = "general")
  () => {
    const connection = createConnection('general');
    connection.connect();
    return () => connection.disconnect();
  },
  // Dependencies for the first render (roomId = "general")
  ['general']
```

React运行这个 Effect，它连接到 `'general'` 聊天室。

#### 用相同的依赖关系重新渲染 {/*re-render-with-same-dependencies*/}

假设 `<ChatRoom roomId="general" />` 重新渲染。JSX输出是一样的：

```js
  // JSX for the second render (roomId = "general")
  return <h1>Welcome to general!</h1>;
```

React看到渲染输出没有改变，所以它没有更新DOM。

第二次渲染的 Effect 看起来是这样的：

```js
  // Effect for the second render (roomId = "general")
  () => {
    const connection = createConnection('general');
    connection.connect();
    return () => connection.disconnect();
  },
  // Dependencies for the second render (roomId = "general")
  ['general']
```

React将第二次渲染中的 `['general']` 与第一次渲染中的 `['general']` 进行比较。**因为所有的依赖关系都是一样的，所以React *忽略* 了第二次渲染的 Effect。** 它从未被调用。

#### 用不同的依赖关系重新渲染 {/*re-render-with-different-dependencies*/}

然后，用户访问 `<ChatRoom roomId="travel" />`。这一次，该组件返回不同的JSX：

```js
  // JSX for the third render (roomId = "travel")
  return <h1>Welcome to travel!</h1>;
```

React更新了DOM，将 `"Welcome to general"` 改为 `"Welcome to travel"`。

第三次渲染的 Effect 看起来是这样的：

```js
  // Effect for the third render (roomId = "travel")
  () => {
    const connection = createConnection('travel');
    connection.connect();
    return () => connection.disconnect();
  },
  // Dependencies for the third render (roomId = "travel")
  ['travel']
```

React将第三次渲染中的 `['travel']` 与第二次渲染中的 `['general']` 进行比较。第一个依赖关系是不同的：`Object.is('travel', 'general')`的结果是 `false` 。所以这个 Effect 不能被跳过。

**在React应用第三次渲染的 Effect 之前，它需要清理最后运行的 Effect。** 第二次渲染的 Effect 被跳过，所以React需要清理第一次渲染的 Effect。如果你向上滚动到第一次渲染，你会看到它的清理工作在用 `createConnection('general')` 创建的连接上调用 `disconnect()` 。这将断开应用程序与 `'general'` 聊天室的连接。

之后，React运行第三次渲染的 Effect。它连接到 `'travel'` 聊天室。

#### 卸载 {/*unmount*/}

最后，让我们假设用户导航离开，`ChatRoom` 组件卸载。React运行最后一个 Effect 的清理函数。最后一个 Effect 是在第三次渲染时产生的。第三次渲染的清理功能破坏了 `createConnection('travel')` 连接。因此，应用程序与 `'travel'` 房间断开连接。

#### 仅开发行为 {/*development-only-behaviors*/}

当 [Strict Mode](/reference/react/StrictMode) 开启时，React在挂载后对每个组件重新挂载一次（状态和DOM被保留）。这可以帮助你找到需要清理的 Effect，并尽早暴露出诸如 race condition 等错误。此外，每当你在开发中保存一个文件时，React都会重新挂载 Effect。这两种行为都是开发专用的。

</DeepDive>

<Recap>

- 与事件不同，效果是由渲染本身引起的，而不是由某个特定的互动引起的。
- 效果可以让你将一个组件与一些外部系统（第三方API，网络等）同步。
- 默认情况下，Effect 在每次渲染后运行（包括初始渲染）。
- 如果它的所有依赖关系的值与上次渲染时相同，React将跳过 Effect。
- 你不能 “选择” 你的依赖关系。它们是由 Effect 里面的代码决定的。
- 一个空的依赖关系数组（ `[]` ）对应于组件的 “mounting”，即被添加到屏幕上。
- 当严格模式开启时，React会对组件进行两次挂载（仅在开发中！）以对你的 Effect 进行压力测试。
- 如果你的 Effect 因为重新挂载而损坏，你需要实现一个清理函数。
- React会在 Effect 下次运行前，以及在卸载时调用你的清理函数。

</Recap>

<Challenges>

#### Focus a field on mount {/*focus-a-field-on-mount*/}

In this example, the form renders a `<MyInput />` component.

Use the input's [`focus()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/focus) method to make `MyInput` automatically focus when it appears on the screen. There is already a commented out implementation, but it doesn't quite work. Figure out why it doesn't work, and fix it. (If you're familiar with the `autoFocus` attribute, pretend that it does not exist: we are reimplementing the same functionality from scratch.)

<Sandpack>

```js MyInput.js active
import { useEffect, useRef } from 'react';

export default function MyInput({ value, onChange }) {
  const ref = useRef(null);

  // TODO: This doesn't quite work. Fix it.
  // ref.current.focus()    

  return (
    <input
      ref={ref}
      value={value}
      onChange={onChange}
    />
  );
}
```

```js App.js hidden
import { useState } from 'react';
import MyInput from './MyInput.js';

export default function Form() {
  const [show, setShow] = useState(false);
  const [name, setName] = useState('Taylor');
  const [upper, setUpper] = useState(false);
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} form</button>
      <br />
      <hr />
      {show && (
        <>
          <label>
            Enter your name:
            <MyInput
              value={name}
              onChange={e => setName(e.target.value)}
            />
          </label>
          <label>
            <input
              type="checkbox"
              checked={upper}
              onChange={e => setUpper(e.target.checked)}
            />
            Make it uppercase
          </label>
          <p>Hello, <b>{upper ? name.toUpperCase() : name}</b></p>
        </>
      )}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>


To verify that your solution works, press "Show form" and verify that the input receives focus (becomes highlighted and the cursor is placed inside). Press "Hide form" and "Show form" again. Verify the input is highlighted again.

`MyInput` should only focus _on mount_ rather than after every render. To verify that the behavior is right, press "Show form" and then repeatedly press the "Make it uppercase" checkbox. Clicking the checkbox should _not_ focus the input above it.

<Solution>

Calling `ref.current.focus()` during render is wrong because it is a *side effect*. Side effects should either be placed inside an event handler or be declared with `useEffect`. In this case, the side effect is _caused_ by the component appearing rather than by any specific interaction, so it makes sense to put it in an Effect.

To fix the mistake, wrap the `ref.current.focus()` call into an Effect declaration. Then, to ensure that this Effect runs only on mount rather than after every render, add the empty `[]` dependencies to it.

<Sandpack>

```js MyInput.js active
import { useEffect, useRef } from 'react';

export default function MyInput({ value, onChange }) {
  const ref = useRef(null);

  useEffect(() => {
    ref.current.focus();
  }, []);

  return (
    <input
      ref={ref}
      value={value}
      onChange={onChange}
    />
  );
}
```

```js App.js hidden
import { useState } from 'react';
import MyInput from './MyInput.js';

export default function Form() {
  const [show, setShow] = useState(false);
  const [name, setName] = useState('Taylor');
  const [upper, setUpper] = useState(false);
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} form</button>
      <br />
      <hr />
      {show && (
        <>
          <label>
            Enter your name:
            <MyInput
              value={name}
              onChange={e => setName(e.target.value)}
            />
          </label>
          <label>
            <input
              type="checkbox"
              checked={upper}
              onChange={e => setUpper(e.target.checked)}
            />
            Make it uppercase
          </label>
          <p>Hello, <b>{upper ? name.toUpperCase() : name}</b></p>
        </>
      )}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

</Solution>

#### Focus a field conditionally {/*focus-a-field-conditionally*/}

This form renders two `<MyInput />` components.

Press "Show form" and notice that the second field automatically gets focused. This is because both of the `<MyInput />` components try to focus the field inside. When you call `focus()` for two input fields in a row, the last one always "wins".

Let's say you want to focus the first field. The first `MyInput` component now receives a boolean `shouldFocus` prop set to `true`. Change the logic so that `focus()` is only called if the `shouldFocus` prop received by `MyInput` is `true`.

<Sandpack>

```js MyInput.js active
import { useEffect, useRef } from 'react';

export default function MyInput({ shouldFocus, value, onChange }) {
  const ref = useRef(null);

  // TODO: call focus() only if shouldFocus is true.
  useEffect(() => {
    ref.current.focus();
  }, []);

  return (
    <input
      ref={ref}
      value={value}
      onChange={onChange}
    />
  );
}
```

```js App.js hidden
import { useState } from 'react';
import MyInput from './MyInput.js';

export default function Form() {
  const [show, setShow] = useState(false);
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');
  const [upper, setUpper] = useState(false);
  const name = firstName + ' ' + lastName;
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} form</button>
      <br />
      <hr />
      {show && (
        <>
          <label>
            Enter your first name:
            <MyInput
              value={firstName}
              onChange={e => setFirstName(e.target.value)}
              shouldFocus={true}
            />
          </label>
          <label>
            Enter your last name:
            <MyInput
              value={lastName}
              onChange={e => setLastName(e.target.value)}
              shouldFocus={false}
            />
          </label>
          <p>Hello, <b>{upper ? name.toUpperCase() : name}</b></p>
        </>
      )}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

To verify your solution, press "Show form" and "Hide form" repeatedly. When the form appears, only the *first* input should get focused. This is because the parent component renders the first input with `shouldFocus={true}` and the second input with `shouldFocus={false}`. Also check that both inputs still work and you can type into both of them.

<Hint>

You can't declare an Effect conditionally, but your Effect can include conditional logic.

</Hint>

<Solution>

Put the conditional logic inside the Effect. You will need to specify `shouldFocus` as a dependency because you are using it inside the Effect. (This means that if some input's `shouldFocus` changes from `false` to `true`, it will focus after mount.)

<Sandpack>

```js MyInput.js active
import { useEffect, useRef } from 'react';

export default function MyInput({ shouldFocus, value, onChange }) {
  const ref = useRef(null);

  useEffect(() => {
    if (shouldFocus) {
      ref.current.focus();
    }
  }, [shouldFocus]);

  return (
    <input
      ref={ref}
      value={value}
      onChange={onChange}
    />
  );
}
```

```js App.js hidden
import { useState } from 'react';
import MyInput from './MyInput.js';

export default function Form() {
  const [show, setShow] = useState(false);
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');
  const [upper, setUpper] = useState(false);
  const name = firstName + ' ' + lastName;
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} form</button>
      <br />
      <hr />
      {show && (
        <>
          <label>
            Enter your first name:
            <MyInput
              value={firstName}
              onChange={e => setFirstName(e.target.value)}
              shouldFocus={true}
            />
          </label>
          <label>
            Enter your last name:
            <MyInput
              value={lastName}
              onChange={e => setLastName(e.target.value)}
              shouldFocus={false}
            />
          </label>
          <p>Hello, <b>{upper ? name.toUpperCase() : name}</b></p>
        </>
      )}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

</Solution>

#### Fix an interval that fires twice {/*fix-an-interval-that-fires-twice*/}

This `Counter` component displays a counter that should increment every second. On mount, it calls [`setInterval`.](https://developer.mozilla.org/en-US/docs/Web/API/setInterval) This causes `onTick` to run every second. The `onTick` function increments the counter.

However, instead of incrementing once per second, it increments twice. Why is that? Find the cause of the bug and fix it.

<Hint>

Keep in mind that `setInterval` returns an interval ID, which you can pass to [`clearInterval`](https://developer.mozilla.org/en-US/docs/Web/API/clearInterval) to stop the interval.

</Hint>

<Sandpack>

```js Counter.js active
import { useState, useEffect } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    function onTick() {
      setCount(c => c + 1);
    }

    setInterval(onTick, 1000);
  }, []);

  return <h1>{count}</h1>;
}
```

```js App.js hidden
import { useState } from 'react';
import Counter from './Counter.js';

export default function Form() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} counter</button>
      <br />
      <hr />
      {show && <Counter />}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

<Solution>

When [Strict Mode](/reference/react/StrictMode) is on (like in the sandboxes on this site), React remounts each component once in development. This causes the interval to be set up twice, and this is why each second the counter increments twice.

However, React's behavior is not the *cause* of the bug: the bug already exists in the code. React's behavior makes the bug more noticeable. The real cause is that this Effect starts a process but doesn't provide a way to clean it up.

To fix this code, save the interval ID returned by `setInterval`, and implement a cleanup function with [`clearInterval`](https://developer.mozilla.org/en-US/docs/Web/API/clearInterval):

<Sandpack>

```js Counter.js active
import { useState, useEffect } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    function onTick() {
      setCount(c => c + 1);
    }

    const intervalId = setInterval(onTick, 1000);
    return () => clearInterval(intervalId);
  }, []);

  return <h1>{count}</h1>;
}
```

```js App.js hidden
import { useState } from 'react';
import Counter from './Counter.js';

export default function App() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} counter</button>
      <br />
      <hr />
      {show && <Counter />}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

In development, React will still remount your component once to verify that you've implemented cleanup well. So there will be a `setInterval` call, immediately followed by `clearInterval`, and `setInterval` again. In production, there will be only one `setInterval` call. The user-visible behavior in both cases is the same: the counter increments once per second.

</Solution>

#### Fix fetching inside an Effect {/*fix-fetching-inside-an-effect*/}

This component shows the biography for the selected person. It loads the biography by calling an asynchronous function `fetchBio(person)` on mount and whenever `person` changes. That asynchronous function returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) which eventually resolves to a string. When fetching is done, it calls `setBio` to display that string under the select box.

<Sandpack>

```js App.js
import { useState, useEffect } from 'react';
import { fetchBio } from './api.js';

export default function Page() {
  const [person, setPerson] = useState('Alice');
  const [bio, setBio] = useState(null);

  useEffect(() => {
    setBio(null);
    fetchBio(person).then(result => {
      setBio(result);
    });
  }, [person]);

  return (
    <>
      <select value={person} onChange={e => {
        setPerson(e.target.value);
      }}>
        <option value="Alice">Alice</option>
        <option value="Bob">Bob</option>
        <option value="Taylor">Taylor</option>
      </select>
      <hr />
      <p><i>{bio ?? 'Loading...'}</i></p>
    </>
  );
}
```

```js api.js hidden
export async function fetchBio(person) {
  const delay = person === 'Bob' ? 2000 : 200;
  return new Promise(resolve => {
    setTimeout(() => {
      resolve('This is ' + person + '’s bio.');
    }, delay);
  })
}

```

</Sandpack>


There is a bug in this code. Start by selecting "Alice". Then select "Bob" and then immediately after that select "Taylor". If you do this fast enough, you will notice that bug: Taylor is selected, but the paragraph below says "This is Bob's bio."

Why does this happen? Fix the bug inside this Effect.

<Hint>

If an Effect fetches something asynchronously, it usually needs cleanup.

</Hint>

<Solution>

To trigger the bug, things need to happen in this order:

- Selecting `'Bob'` triggers `fetchBio('Bob')`
- Selecting `'Taylor'` triggers `fetchBio('Taylor')`
- **Fetching `'Taylor'` completes *before* fetching `'Bob'`**
- The Effect from the `'Taylor'` render calls `setBio('This is Taylor’s bio')`
- Fetching `'Bob'` completes
- The Effect from the `'Bob'` render calls `setBio('This is Bob’s bio')`

This is why you see Bob's bio even though Taylor is selected. Bugs like this are called [race conditions](https://en.wikipedia.org/wiki/Race_condition) because two asynchronous operations are "racing" with each other, and they might arrive in an unexpected order.

To fix this race condition, add a cleanup function:

<Sandpack>

```js App.js
import { useState, useEffect } from 'react';
import { fetchBio } from './api.js';

export default function Page() {
  const [person, setPerson] = useState('Alice');
  const [bio, setBio] = useState(null);
  useEffect(() => {
    let ignore = false;
    setBio(null);
    fetchBio(person).then(result => {
      if (!ignore) {
        setBio(result);
      }
    });
    return () => {
      ignore = true;
    }
  }, [person]);

  return (
    <>
      <select value={person} onChange={e => {
        setPerson(e.target.value);
      }}>
        <option value="Alice">Alice</option>
        <option value="Bob">Bob</option>
        <option value="Taylor">Taylor</option>
      </select>
      <hr />
      <p><i>{bio ?? 'Loading...'}</i></p>
    </>
  );
}
```

```js api.js hidden
export async function fetchBio(person) {
  const delay = person === 'Bob' ? 2000 : 200;
  return new Promise(resolve => {
    setTimeout(() => {
      resolve('This is ' + person + '’s bio.');
    }, delay);
  })
}

```

</Sandpack>

Each render's Effect has its own `ignore` variable. Initially, the `ignore` variable is set to `false`. However, if an Effect gets cleaned up (such as when you select a different person), its `ignore` variable becomes `true`. So now it doesn't matter in which order the requests complete. Only the last person's Effect will have `ignore` set to `false`, so it will call `setBio(result)`. Past Effects have been cleaned up, so the `if (!ignore)` check will prevent them from calling `setBio`:

- Selecting `'Bob'` triggers `fetchBio('Bob')`
- Selecting `'Taylor'` triggers `fetchBio('Taylor')` **and cleans up the previous (Bob's) Effect**
- Fetching `'Taylor'` completes *before* fetching `'Bob'`
- The Effect from the `'Taylor'` render calls `setBio('This is Taylor’s bio')`
- Fetching `'Bob'` completes
- The Effect from the `'Bob'` render **does not do anything because its `ignore` flag was set to `true`**

In addition to ignoring the result of an outdated API call, you can also use [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) to cancel the requests that are no longer needed. However, by itself this is not enough to protect against race conditions. More asynchronous steps could be chained after the fetch, so using an explicit flag like `ignore` is the most reliable way to fix this type of problems.

</Solution>

</Challenges>

