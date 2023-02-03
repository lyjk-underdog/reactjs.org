---
title: '用 ref 来引用数值'
---

<Intro>

当你想让一个组件 “记住” 一些信息，但又不想让这些信息[触发新的渲染](/learn/render-and-commit)时，你可以使用一个*ref*。

</Intro>

<YouWillLearn>

- 如何在你的组件中添加一个 ref
- 如何更新一个 ref 的值
- ref 与 state 有什么不同
- 如何安全地使用 ref

</YouWillLearn>

## 在你的组件中添加一个 ref {/*adding-a-ref-to-your-component*/}

你可以通过从React中导入 `useRef` Hook来给你的组件添加一个 ref ：

```js
import { useRef } from 'react';
```

在你的组件中，调用 `useRef` Hook，并将你想要引用的初始值作为唯一的参数传递给它。例如，这里是对值 `0` 的引用：

```js
const ref = useRef(0);
```

`useRef` 返回一个像这样的对象：

```js
{ 
  current: 0 // The value you passed to useRef
}
```

<Illustration src="/images/docs/illustrations/i_ref.png" alt="An arrow with 'current' written on it stuffed into a pocket with 'ref' written on it." />

你可以通过 `ref.current` 属性访问该 ref 的当前值。这个值故意是可变的，这意味着你可以读取和写入它。它就像你的组件的一个秘密口袋，React并不跟踪它。(这就是为什么它是React单向数据流的 "逃生舱"--下面会详细介绍！)

这里，一个按钮将在每次点击时增加 `ref.current` ：

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

ref 指向一个数字，但是和 [state](/learn/state-a-components-memory) 一样，你可以指向任何东西：一个字符串，一个对象，甚至是一个函数。与 state 不同，ref 是一个普通的JavaScript对象，你可以可以读取和修改它的 `current` 属性。

请注意，**该组件不会在每次增量时重新渲染**。像 state 一样，ref 在重新渲染之间被React保留。然而，设置 state 会重新渲染一个组件，而改变 ref 则不会！

## 案例：建立一个秒表 {/*example-building-a-stopwatch*/}

你可以在一个组件中结合 ref 和 state 。例如，让我们做一个秒表，用户可以通过按一个按钮开始或停止。为了显示自用户按下 “开始” 以来已经过去了多少时间，你将需要跟踪 “开始” 按钮被按下的时间以及当前的时间是多少。**这个信息是用来渲染的，所以你要保持它的状态：**

```js
const [startTime, setStartTime] = useState(null);
const [now, setNow] = useState(null);
```

当用户按下 “开始” 时，你将使用 [`setInterval`](https://developer.mozilla.org/docs/Web/API/setInterval)，以便每10毫秒更新一次时间：

<Sandpack>

```js
import { useState } from 'react';

export default function Stopwatch() {
  const [startTime, setStartTime] = useState(null);
  const [now, setNow] = useState(null);

  function handleStart() {
    // Start counting.
    setStartTime(Date.now());
    setNow(Date.now());

    setInterval(() => {
      // Update the current time every 10ms.
      setNow(Date.now());
    }, 10);
  }

  let secondsPassed = 0;
  if (startTime != null && now != null) {
    secondsPassed = (now - startTime) / 1000;
  }

  return (
    <>
      <h1>Time passed: {secondsPassed.toFixed(3)}</h1>
      <button onClick={handleStart}>
        Start
      </button>
    </>
  );
}
```

</Sandpack>

当 “停止” 按钮被按下时，你需要取消现有的 interval，使其停止更新 `now` 状态变量。你可以通过调用[`clearInterval`](https://developer.mozilla.org/en-US/docs/Web/API/clearInterval)来做到这一点，但是你需要给它一个 interval ID ，这个 interval ID 是之前在用户按下 Start 时通过 `setInterval` 返回的。你需要在某个地方保留这个 interval ID。由于 interval ID 不用于渲染，你可以将它保存在一个 ref 中。

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function Stopwatch() {
  const [startTime, setStartTime] = useState(null);
  const [now, setNow] = useState(null);
  const intervalRef = useRef(null);

  function handleStart() {
    setStartTime(Date.now());
    setNow(Date.now());

    clearInterval(intervalRef.current);
    intervalRef.current = setInterval(() => {
      setNow(Date.now());
    }, 10);
  }

  function handleStop() {
    clearInterval(intervalRef.current);
  }

  let secondsPassed = 0;
  if (startTime != null && now != null) {
    secondsPassed = (now - startTime) / 1000;
  }

  return (
    <>
      <h1>Time passed: {secondsPassed.toFixed(3)}</h1>
      <button onClick={handleStart}>
        Start
      </button>
      <button onClick={handleStop}>
        Stop
      </button>
    </>
  );
}
```

</Sandpack>

当一个信息被用于渲染时，要保持它的状态。当一个信息只被事件处理程序所需要，并且改变它不需要重新渲染时，使用 ref 可能更有效率。

## ref 与 state 之间的区别 {/*differences-between-refs-and-state*/}

也许你认为 ref 似乎没有 state 那么 “严格”——你可以对它们进行突变，而不是总是要使用状态设置函数。但在大多数情况下，你应该使用 state，ref 是一个你不常需要的 “逃生舱”。下面是 state 和 ref 的比较：

| ref                                                     | state                                                                                                |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `useRef(initialValue)` 返回 `{ current: initialValue }` | `useState(initialValue)` 返回一个状态变量的当前值和一个状态设置函数 ( `[value, setValue]`)           |
| 当你改变它时，不会触发重新渲染。                        | 当你改变它时，会触发重新渲染。                                                                       |
| 可变的——你可以在渲染过程之外修改和更新 `current` 的值。 | "不可改变"——你必须使用状态设置函数来修改状态变量以排队重新渲染。                                     |
| 在渲染过程中，你不应该读取（或写入）当前值。            | 你可以在任何时候读取状态。然而，每次渲染都有自己的状态[快照](/learn/state-as-a-snapshot)，不会改变。 |

这里有一个用 state 实现的计数器按钮：

<Sandpack>

```js
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <button onClick={handleClick}>
      You clicked {count} times
    </button>
  );
}
```

</Sandpack>

因为 `count` 值是需要显示的，所以为它使用一个状态值是合理的。当计数器的值被 `setCount()` 设置后，React会重新渲染该组件，屏幕会更新以反映新的计数。

如果你试图用 ref 来实现这一点，React将永远不会重新渲染组件，所以你永远不会看到计数的变化！点击这个按钮，看看它**为什么不更新文本**：

<Sandpack>

```js
import { useRef } from 'react';

export default function Counter() {
  let countRef = useRef(0);

  function handleClick() {
    // This doesn't re-render the component!
    countRef.current = countRef.current + 1;
  }

  return (
    <button onClick={handleClick}>
      You clicked {countRef.current} times
    </button>
  );
}
```

</Sandpack>

这就是为什么在渲染过程中读取 `ref.current` 会导致不可靠的代码。如果你需要，就用 state 代替。

<DeepDive>

#### useRef 在内部是如何工作的？ {/*how-does-use-ref-work-inside*/}

尽管 `useState` 和 `useRef` 都是由React提供的，但原则上 `useRef` 可以在 `useState` 的基础上实现。你可以想象，在React内部，`useRef` 是这样实现的：

```js
// Inside of React
function useRef(initialValue) {
  const [ref, unused] = useState({ current: initialValue });
  return ref;
}
```

在第一次渲染时，`useRef` 返回 `{ current: initialValue }`。这个对象被React存储起来，所以在下一次渲染时，会返回同一个对象。请注意，在这个例子中，状态设置器是未使用的。它是不必要的，因为 `useRef` 总是需要返回相同的对象!

React提供了 `useRef` 的内置版本，因为它在实践中足够常见。但是你可以把它看成是一个没有 setter 的普通状态变量。如果你熟悉面向对象编程，ref 可能会让你想起实例字段——但你写的不是`this.something`，而是`somethingRef.current`。

</DeepDive>

## 什么时候使用 ref {/*when-to-use-refs*/}

通常情况下，当你的组件需要 “走出” React并与外部API通信时，你会使用 ref，通常是不会影响组件外观的浏览器API。下面是一些罕见的情况：

- 存储 [timeout ID](https://developer.mozilla.org/docs/Web/API/setTimeout)
- 存储和操作 [DOM elements](https://developer.mozilla.org/docs/Web/API/Element)，我们将在 [下一页](/learn/manipulating-the-dom-with-refs)介绍
- 储存其他对象，这些对象对于计算 JSX 是不必要的。

如果你的组件需要存储一些值，但它不影响渲染逻辑，可以选择 ref 。

## ref 最佳实践 {/*best-practices-for-refs*/}

遵循这些原则将使你的组件更加可预测：

- **把 ref 当作一个逃生舱口。** 当你与外部系统或浏览器API一起工作时，ref 是很有用的。如果你的大部分应用逻辑和数据流都依赖于 ref，你可能要重新考虑你的方法。
- **在渲染过程中，不要读或写 `ref.current`。** 如果在渲染过程中需要一些信息，请使用 state 来代替。因为React不知道 `ref.current` 何时改变，即使在渲染时读取它也会使你的组件的行为难以预测。(唯一的例外是像 `if (!ref.current) ref.current = new Thing()` 这样的代码，它只在第一次渲染时设置一次 ref。)

React state 的限制并不适用于 ref。例如，state 在每次[渲染时就像一个快照](/learn/state-as-a-snapshot)，[不会同步更新](/learn/queueing-a-series-of-state-updates)。但是当你突变一个ref 的当前值时，它会立即改变：

```js
ref.current = 5;
console.log(ref.current); // 5
```

这是因为 **ref 本身是一个普通的JavaScript对象，** 所以它的行为也像一个对象。

当你使用 ref 时，你也不需要担心如何[避免变异](/learn/updating-objects-in-state)。只要你要变异的对象不用于渲染，React就不会关心你对引用或其内容做什么。

## ref 与 DOM {/*refs-and-the-dom*/}

你可以将一个 ref 指向任何值。然而，ref 最常见的使用情况是访问一个DOM元素。例如，如果你想以编程方式关注一个输入，这就很方便。当你在JSX中传递一个 `ref` 属性，比如 `<div ref={myRef}>`，React会把相应的DOM元素放入 `myRef.current`。你可以在[用 ref 操纵DOM](/learn/manipulating-the-dom-with-refs)中阅读更多相关内容。

<Recap>

- ref 是用来保存不用于渲染的值的逃生舱口。你不会经常需要它们。
- ref 是一个普通的JavaScript对象，它有一个可以读写的 `current` 的属性。
- 你可以通过调用 `useRef` Hook来要求React给你一个ref。
- 和 state 一样，ref 可以让你在组件的重新渲染之间保留信息。
- 与 state 不同，设置 ref 的 `current` 值不会触发重新渲染。
- 在渲染过程中不要读写 `ref.current` 。这使得你的组件难以预测。

</Recap>



<Challenges>

#### 修复一个破损的聊天输入 {/*fix-a-broken-chat-input*/}

输入一个信息，然后点击 “发送”。你会注意到，在你看到 “已发送！” 的提醒之前，会有三秒钟的延迟。在这个延迟期间，你可以看到一个 “撤销” 按钮。点击它。这个 “撤销” 按钮应该是为了阻止 “已发送！” 信息的出现。它通过调用 `handleSend` 过程中保存的 timeout ID 的 [`clearTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/clearTimeout) 来实现这一目的。然而，即使点击了 “撤销”， “已发送！” 信息仍然出现。找到它不工作的原因，并加以解决。

<Hint>

像 `let timeoutID` 这样的常规变量不会在重新渲染之间 "存活"，因为每次渲染都会从头开始运行你的组件（并初始化其变量）。你应该把 timeout ID 放在别的地方吗？

</Hint>

<Sandpack>

```js
import { useState } from 'react';

export default function Chat() {
  const [text, setText] = useState('');
  const [isSending, setIsSending] = useState(false);
  let timeoutID = null;

  function handleSend() {
    setIsSending(true);
    timeoutID = setTimeout(() => {
      alert('Sent!');
      setIsSending(false);
    }, 3000);
  }

  function handleUndo() {
    setIsSending(false);
    clearTimeout(timeoutID);
  }

  return (
    <>
      <input
        disabled={isSending}
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <button
        disabled={isSending}
        onClick={handleSend}>
        {isSending ? 'Sending...' : 'Send'}
      </button>
      {isSending &&
        <button onClick={handleUndo}>
          Undo
        </button>
      }
    </>
  );
}
```

</Sandpack>

<Solution>

每当你的组件重新出现时（比如你设置状态时），所有的局部变量都会从头开始初始化。这就是为什么你不能把 timeout ID 保存在一个像 `timeoutID` 这样的局部变量中，然后期望另一个事件处理程序在未来 “看到” 它。相反，你可以把它存储在一个 ref 中，React 会在两次渲染之间保留它。

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function Chat() {
  const [text, setText] = useState('');
  const [isSending, setIsSending] = useState(false);
  const timeoutRef = useRef(null);

  function handleSend() {
    setIsSending(true);
    timeoutRef.current = setTimeout(() => {
      alert('Sent!');
      setIsSending(false);
    }, 3000);
  }

  function handleUndo() {
    setIsSending(false);
    clearTimeout(timeoutRef.current);
  }

  return (
    <>
      <input
        disabled={isSending}
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <button
        disabled={isSending}
        onClick={handleSend}>
        {isSending ? 'Sending...' : 'Send'}
      </button>
      {isSending &&
        <button onClick={handleUndo}>
          Undo
        </button>
      }
    </>
  );
}
```

</Sandpack>

</Solution>


#### 修复一个组件无法重新渲染的问题 {/*fix-a-component-failing-to-re-render*/}

这个按钮应该是在显示 “On” 和 “Off” 之间切换的。然而，它总是显示 “Off”。这段代码有什么问题？修复它。

<Sandpack>

```js
import { useRef } from 'react';

export default function Toggle() {
  const isOnRef = useRef(false);

  return (
    <button onClick={() => {
      isOnRef.current = !isOnRef.current;
    }}>
      {isOnRef.current ? 'On' : 'Off'}
    </button>
  );
}
```

</Sandpack>

<Solution>

在这个例子中，一个 ref 的当前值被用来计算渲染输出：`{isOnRef.current ? 'On' : 'Off'}`。这是一个迹象，表明这些信息不应该在 ref 中出现，而应该放在 state 中。要解决这个问题，请删除 ref，而使用 state：

<Sandpack>

```js
import { useState } from 'react';

export default function Toggle() {
  const [isOn, setIsOn] = useState(false);

  return (
    <button onClick={() => {
      setIsOn(!isOn);
    }}>
      {isOn ? 'On' : 'Off'}
    </button>
  );
}
```

</Sandpack>

</Solution>

#### 修复防抖问题 {/*fix-debouncing*/}

在这个例子中，所有的按钮点击处理程序都是 [“防抖”](https://redd.one/blog/debounce-vs-throttle) 的。要看看这意味着什么，请按下其中一个按钮。注意一秒钟后消息是如何出现的。如果你在等待消息时按下按钮，计时器会重置。因此，如果你一直快速点击同一个按钮很多次，那么在你停止点击一秒钟*后*，信息才会出现。防抖让你延迟一些动作，直到用户 “停止做事”。

这个例子可以工作，但不太符合预期。这些按钮不是独立的。要看到这个问题，点击其中一个按钮，然后立即点击另一个按钮。你会认为在延迟之后，会看到两个按钮的信息。但只有最后一个按钮的信息显示出来。第一个按钮的信息会丢失。

为什么按钮之间会相互干扰？找到并解决这个问题。

<Hint>

所有 `DebouncedButton` 组件之间共享最后的 timeout ID 变量。这就是为什么点击一个按钮会重置另一个按钮的超时。你能为每个按钮存储一个单独的 timeout ID 吗？

</Hint>

<Sandpack>

```js
import { useState } from 'react';

let timeoutID;

function DebouncedButton({ onClick, children }) {
  return (
    <button onClick={() => {
      clearTimeout(timeoutID);
      timeoutID = setTimeout(() => {
        onClick();
      }, 1000);
    }}>
      {children}
    </button>
  );
}

export default function Dashboard() {
  return (
    <>
      <DebouncedButton
        onClick={() => alert('Spaceship launched!')}
      >
        Launch the spaceship
      </DebouncedButton>
      <DebouncedButton
        onClick={() => alert('Soup boiled!')}
      >
        Boil the soup
      </DebouncedButton>
      <DebouncedButton
        onClick={() => alert('Lullaby sung!')}
      >
        Sing a lullaby
      </DebouncedButton>
    </>
  )
}
```

```css
button { display: block; margin: 10px; }
```

</Sandpack>

<Solution>

像 `timeoutID` 这样的变量是在所有组件之间共享的。这就是为什么点击第二个按钮会重置第一个按钮的待定超时。为了解决这个问题，你可以把 timeout ID 放在一个 ref 中。每个按钮将得到它自己的 ref，所以它们不会互相冲突。请注意，快速点击两个按钮将显示两个消息。

<Sandpack>

```js
import { useState, useRef } from 'react';

function DebouncedButton({ onClick, children }) {
  const timeoutRef = useRef(null);
  return (
    <button onClick={() => {
      clearTimeout(timeoutRef.current);
      timeoutRef.current = setTimeout(() => {
        onClick();
      }, 1000);
    }}>
      {children}
    </button>
  );
}

export default function Dashboard() {
  return (
    <>
      <DebouncedButton
        onClick={() => alert('Spaceship launched!')}
      >
        Launch the spaceship
      </DebouncedButton>
      <DebouncedButton
        onClick={() => alert('Soup boiled!')}
      >
        Boil the soup
      </DebouncedButton>
      <DebouncedButton
        onClick={() => alert('Lullaby sung!')}
      >
        Sing a lullaby
      </DebouncedButton>
    </>
  )
}
```

```css
button { display: block; margin: 10px; }
```

</Sandpack>

</Solution>

#### 读取最新的状态 {/*read-the-latest-state*/}

在这个例子中，在你按下 “Send” 之后，在信息显示之前会有一个小的延迟。输入 “hello”，按 “Send”，然后再次快速编辑输入。尽管你做了编辑，警报仍然会显示 “hello”（这是在[点击按钮时](/learn/state-as-a-snapshot#state-over-time)的状态值）。

通常情况下，这种行为是你在应用程序中所希望的。然而，偶尔也会有这样的情况，你想用一些异步代码来读取一些状态的*最新*版本。你能想到一种方法，使警报显示*当前*的输入文本，而不是点击时的文本吗？

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function Chat() {
  const [text, setText] = useState('');

  function handleSend() {
    setTimeout(() => {
      alert('Sending: ' + text);
    }, 3000);
  }

  return (
    <>
      <input
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <button
        onClick={handleSend}>
        Send
      </button>
    </>
  );
}
```

</Sandpack>

<Solution>

State works [like a snapshot](/learn/state-as-a-snapshot), so you can't read the latest state from an asynchronous operation like a timeout. However, you can keep the latest input text in a ref. A ref is mutable, so you can read the `current` property at any time. Since the current text is also used for rendering, in this example, you will need *both* a state variable (for rendering), *and* a ref (to read it in the timeout). You will need to update the current ref value manually.

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function Chat() {
  const [text, setText] = useState('');
  const textRef = useRef(text);

  function handleChange(e) {
    setText(e.target.value);
    textRef.current = e.target.value;
  }

  function handleSend() {
    setTimeout(() => {
      alert('Sending: ' + textRef.current);
    }, 3000);
  }

  return (
    <>
      <input
        value={text}
        onChange={handleChange}
      />
      <button
        onClick={handleSend}>
        Send
      </button>
    </>
  );
}
```

</Sandpack>

</Solution>

</Challenges>
