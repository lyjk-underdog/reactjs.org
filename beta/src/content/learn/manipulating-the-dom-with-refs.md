---
title: '用 ref 操纵DOM'
---

<Intro>

React会自动更新[DOM](https://developer.mozilla.org/docs/Web/API/Document_Object_Model/Introduction)以匹配你的渲染输出，所以你的组件不会经常需要操作它。然而，有时你可能需要访问由React管理的DOM元素--例如，聚焦一个节点，滚动到它，或测量它的大小和位置。在React中没有内置的方法来做这些事情，所以你将需要一个指向DOM节点的 *ref* 。

</Intro>

<YouWillLearn>

- 如何用JSX的 `ref` 属性访问由React管理的DOM节点
- JSX的`ref` 属性与 `useRef` Hook的关系
- 如何访问另一个组件的DOM节点
- 在哪些情况下，修改React管理的DOM是安全的

</YouWillLearn>

## 获取节点的引用 {/*getting-a-ref-to-the-node*/}

要访问一个由React管理的DOM节点，首先要导入 `useRef` Hook：

```js
import { useRef } from 'react';
```

然后，用它来声明你的组件中的一个 ref ：

```js
const myRef = useRef(null);
```

最后，将其作为 `ref` 属性传递给DOM节点：

```js
<div ref={myRef}>
```

`useRef` Hook会返回一个具有单一属性的对象，该属性名为 `current` 。最初，`myRef.current` 将为 `null` 。当React为这个 `<div>` 创建一个DOM节点时，React会把这个节点的引用放入 `myRef.current` 。然后你可以从你的[事件处理程序](/learn/responding-to-events)中访问这个DOM节点，并使用定义在它上面的内置[浏览器API](https://developer.mozilla.org/docs/Web/API/Element)。

```js
// You can use any browser APIs, for example:
myRef.current.scrollIntoView();
```

### 案例：聚焦一个文本输入 {/*example-focusing-a-text-input*/}

在这个例子中，点击按钮将聚焦输入：

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

为了实现这一功能：

1. 用 `useRef` Hook声明 `inputRef`。
2. 将其作为 `<input ref={inputRef}>` 传递。这告诉React**把这个 `<input>` 的DOM节点放入 `inputRef.current` 。**
3. 在 `handleClick` 函数中，从 `inputRef.current` 中读取输入的DOM节点，并用 `inputRef.current.focus()` 对其调用[`focus()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/focus)方法。
4. 将 `handleClick` 事件处理程序绑定到 `<button>` 的 `onClick` 事件上 。

虽然DOM操作是 ref 最常见的用例，但 `useRef` Hook也可以用来存储React之外的其他东西，比如 timer ID。与 state 类似，ref 在渲染之间保持不变，但是与 state 不同的是，当你设置它们时不会触发重新渲染。关于 ref 的介绍，请看 [用 ref 来引用数值](/learn/referencing-values-with-refs) 一章。

### 案例：滚动到一个元素 {/*example-scrolling-to-an-element*/}

你可以在一个组件中拥有多于一个的 ref 。在这个例子中，有一个三张图片的幻灯片。每个按钮都通过调用相应的DOM节点的[ `scrollIntoView()` ](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView)方法将一个图片置于中心位置：

<Sandpack>

```js
import { useRef } from 'react';

export default function CatFriends() {
  const firstCatRef = useRef(null);
  const secondCatRef = useRef(null);
  const thirdCatRef = useRef(null);

  function handleScrollToFirstCat() {
    firstCatRef.current.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest',
      inline: 'center'
    });
  }

  function handleScrollToSecondCat() {
    secondCatRef.current.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest',
      inline: 'center'
    });
  }

  function handleScrollToThirdCat() {
    thirdCatRef.current.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest',
      inline: 'center'
    });
  }

  return (
    <>
      <nav>
        <button onClick={handleScrollToFirstCat}>
          Tom
        </button>
        <button onClick={handleScrollToSecondCat}>
          Maru
        </button>
        <button onClick={handleScrollToThirdCat}>
          Jellylorum
        </button>
      </nav>
      <div>
        <ul>
          <li>
            <img
              src="https://placekitten.com/g/200/200"
              alt="Tom"
              ref={firstCatRef}
            />
          </li>
          <li>
            <img
              src="https://placekitten.com/g/300/200"
              alt="Maru"
              ref={secondCatRef}
            />
          </li>
          <li>
            <img
              src="https://placekitten.com/g/250/200"
              alt="Jellylorum"
              ref={thirdCatRef}
            />
          </li>
        </ul>
      </div>
    </>
  );
}
```

```css
div {
  width: 100%;
  overflow: hidden;
}

nav {
  text-align: center;
}

button {
  margin: .25rem;
}

ul,
li {
  list-style: none;
  white-space: nowrap;
}

li {
  display: inline;
  padding: 0.5rem;
}
```

</Sandpack>

<DeepDive>

#### 如何使用 ref 回调来管理一系列的 ref {/*how-to-manage-a-list-of-refs-using-a-ref-callback*/}

在上面的例子中，我们是提前知道 ref 的数量的。然而，有时你可能需要对列表中的每一个项目绑定 ref ，而你不知道你会有多少个。像这样的东西就**不能用** 了：

```js
<ul>
  {items.map((item) => {
    // Doesn't work!
    const ref = useRef(null);
    return <li ref={ref} />;
  })}
</ul>
```

这是因为 Hook 必须**只在你的组件的顶层调用**。你不能在一个循环中、在一个条件中或在一个 `map()` 调用中调用 `useRef`。

一个可能的方法是获得他们的父元素的单一引用，然后使用DOM操作方法，如[ `querySelectorAll` ](https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelectorAll)，从中 “找到” 单个子节点。然而，这是很脆弱的，如果你的DOM结构发生变化，就会出现问题。

另一个解决方案是**向 `ref` 属性传递一个函数**。这被称为[`ref` callback](/reference/react-dom/components/common#ref-callback)。React 会在设置 ref 的时候用 DOM 节点调用你的 ref 回调，而在清除 ref 的时候则用 `null`。这让你可以维护你自己的数组或[ Map ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map)，并通过其索引或某种ID来访问任何引用。

这个例子显示了如何使用这种方法来滚动到长列表中的任意节点：

<Sandpack>

```js
import { useRef } from 'react';

export default function CatFriends() {
  const itemsRef = useRef(null);

  function scrollToId(itemId) {
    const map = getMap();
    const node = map.get(itemId);
    node.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest',
      inline: 'center'
    });
  }

  function getMap() {
    if (!itemsRef.current) {
      // Initialize the Map on first usage.
      itemsRef.current = new Map();
    }
    return itemsRef.current;
  }

  return (
    <>
      <nav>
        <button onClick={() => scrollToId(0)}>
          Tom
        </button>
        <button onClick={() => scrollToId(5)}>
          Maru
        </button>
        <button onClick={() => scrollToId(9)}>
          Jellylorum
        </button>
      </nav>
      <div>
        <ul>
          {catList.map(cat => (
            <li
              key={cat.id}
              ref={(node) => {
                const map = getMap();
                if (node) {
                  map.set(cat.id, node);
                } else {
                  map.delete(cat.id);
                }
              }}
            >
              <img
                src={cat.imageUrl}
                alt={'Cat #' + cat.id}
              />
            </li>
          ))}
        </ul>
      </div>
    </>
  );
}

const catList = [];
for (let i = 0; i < 10; i++) {
  catList.push({
    id: i,
    imageUrl: 'https://placekitten.com/250/200?image=' + i
  });
}

```

```css
div {
  width: 100%;
  overflow: hidden;
}

nav {
  text-align: center;
}

button {
  margin: .25rem;
}

ul,
li {
  list-style: none;
  white-space: nowrap;
}

li {
  display: inline;
  padding: 0.5rem;
}
```

</Sandpack>

在这个例子中，`itemsRef` 并没有持有一个单独的 DOM 节点。相反，它持有一个从项目 ID 到 DOM 节点的 [Map](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Map)。([ref 可以持有任何值！](/learn/referencing-values-with-refs))每个列表项的 ref 回调都会注意更新Map：

```js
<li
  key={cat.id}
  ref={node => {
    const map = getMap();
    if (node) {
      // Add to the Map
      map.set(cat.id, node);
    } else {
      // Remove from the Map
      map.delete(cat.id);
    }
  }}
>
```

这让你以后可以从 Map 中读取单个 DOM 节点。

</DeepDive>

## 访问另一个组件的 DOM 节点 {/*accessing-another-components-dom-nodes*/}

当你在一个输出浏览器元素（如 `<input />` ）的内置组件上放一个 ref，React将把该 ref 的 `current` 属性设置为相应的 DOM 节点（如浏览器中实际的 `<input />` ）。

然而，如果你试图在**你自己**的组件上放一个 ref，比如 `<MyInput />` ，默认情况下你会得到 `null` 。下面是一个演示的例子。请注意，点击按钮**并不能**聚焦输入：

<Sandpack>

```js
import { useRef } from 'react';

function MyInput(props) {
  return <input {...props} />;
}

export default function MyForm() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput ref={inputRef} />
      <button onClick={handleClick}>
        Focus the input
      </button>
    </>
  );
}
```

</Sandpack>

为了帮助你注意到这个问题，React也会向控制台打印出一个错误：

<ConsoleBlock level="error">

Warning: Function components cannot be given refs. Attempts to access this ref will fail. Did you mean to use React.forwardRef()?

</ConsoleBlock>

发生这种情况是因为默认情况下，React不允许一个组件访问其他组件的 DOM 节点。即使是它自己的孩子也不行。这是故意的。ref 是一个逃生舱口，应该少用。手动操作其他组件的 DOM 节点会使你的代码更加脆弱。

相反，_想要_ 暴露其DOM节点的组件必须**选择**这种行为：一个组件可以指定把它的 ref “转发” 给它的一个子节点。想要实现这种行为，我们可以使用 `forwardRef` API。下面是 `MyInput` 使用 `forwardRef`  API的例子：

```js
const MyInput = forwardRef((props, ref) => {
  return <input {...props} ref={ref} />;
});
```

这是它的工作流程：

1. `<MyInput ref={inputRef} />` 告诉React将相应的 DOM 节点放入 `inputRef.current` 。然而，这取决于 `MyInput` 组件的选择——默认情况下，它没有选择。
2. `MyInput` 组件是用 `forwardRef` 声明的。**这使它选择接收上面的 `inputRef` 作为第二个 ref 参数**，这个参数是在 props 之后声明的。
3. `MyInput` 本身将它收到的 `ref` 传递给它里面的 `<input>` 。

现在，点击按钮来聚焦输入就可以工作了：

<Sandpack>

```js
import { forwardRef, useRef } from 'react';

const MyInput = forwardRef((props, ref) => {
  return <input {...props} ref={ref} />;
});

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput ref={inputRef} />
      <button onClick={handleClick}>
        Focus the input
      </button>
    </>
  );
}
```

</Sandpack>

在设计系统中，像按钮、输入等低级组件的常见模式是将它们的 ref 转发给它们的 DOM 节点。另一方面，像表单、列表或页面部分这样的高级组件通常不会暴露它们的 DOM 节点，以避免对 DOM 结构的意外依赖。

<DeepDive>

#### 用 `useImperativeHandle` 暴露一个API的子集 {/*exposing-a-subset-of-the-api-with-an-imperative-handle*/}

在上面的例子中，`MyInput` 暴露了原始的 DOM 输入元素。这让父级组件对它调用 `focus()` 。然而，这也让父组件可以做一些其他的事情--例如，改变它的 CSS 样式。在特殊情况下，你可能想要限制暴露的功能。这时候你就可以用 `useImperativeHandle` 做到这一点：

<Sandpack>

```js
import {
  forwardRef, 
  useRef, 
  useImperativeHandle
} from 'react';

const MyInput = forwardRef((props, ref) => {
  const realInputRef = useRef(null);
  useImperativeHandle(ref, () => ({
    // Only expose focus and nothing else
    focus() {
      realInputRef.current.focus();
    },
  }));
  return <input {...props} ref={realInputRef} />;
});

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput ref={inputRef} />
      <button onClick={handleClick}>
        Focus the input
      </button>
    </>
  );
}
```

</Sandpack>

这里，`MyInput` 里面的 `realInputRef` 持有实际的 input DOM 节点的引用。然而，`useImperativeHandle` 指示React提供你自己的特殊对象作为到父组件的 ref 值。所以 `Form` 组件内的 `inputRef.current` 将只有 `focus` 方法。在这种情况下，ref 绑定的不是DOM节点，而是你在 `useImperativeHandle` 调用中创建的自定义对象。

</DeepDive>

## React 什么时候为 ref 绑定 DOM {/*when-react-attaches-the-refs*/}

在React中，每次更新都被分成[两个阶段](/learn/render-and-commit#step-3-react-commits-changes-to-the-dom)：

* 在**渲染**过程中，React调用你的组件来确定屏幕上应该出现什么。
* 在**提交**过程中，React将变化应用到 DOM 中。

一般来说，你[不希望](/learn/referencing-values-with-refs#best-practices-for-refs)在渲染过程中访问 ref ，这对持有 DOM 节点的 ref 也是如此。在第一次渲染时，DOM 节点还没有被创建，所以 `ref.current` 将为 `null` 。而在更新的渲染过程中，DOM 节点还没有被更新，所以现在读取它们也还为时过早。

React在提交过程中设置 `ref.current` 。在更新DOM之前，React将受影响的 `ref.current` 值设置为 `null` 。更新DOM后，React立即将它们设置为相应的 DOM 节点。

**通常情况下，你会从事件处理程序中访问 ref 。** 如果你想对一个 ref 做一些事情，但没有特别的事件可以触发，这时候你可能需要一个 Effect。我们将在接下来的几页中讨论 Effect。

<DeepDive>

#### 用 `flushSync` 同步地更新状态 {/*flushing-state-updates-synchronously-with-flush-sync*/}

考虑一下这样的代码，它添加了一个新的 todo ，并将屏幕向下滚动到列表的最后一个 task 。请注意，由于某些原因，它总是滚动到在最后一个添加的 task 之前的那个 task ：

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function TodoList() {
  const listRef = useRef(null);
  const [text, setText] = useState('');
  const [todos, setTodos] = useState(
    initialTodos
  );

  function handleAdd() {
    const newTodo = { id: nextId++, text: text };
    setText('');
    setTodos([ ...todos, newTodo]);
    listRef.current.lastChild.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest'
    });
  }

  return (
    <>
      <button onClick={handleAdd}>
        Add
      </button>
      <input
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <ul ref={listRef}>
        {todos.map(todo => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
    </>
  );
}

let nextId = 0;
let initialTodos = [];
for (let i = 0; i < 20; i++) {
  initialTodos.push({
    id: nextId++,
    text: 'Todo #' + (i + 1)
  });
}
```

</Sandpack>

问题出在这两行上：

```js
setTodos([ ...todos, newTodo]);
listRef.current.lastChild.scrollIntoView();
```

在React中，[状态更新是排队的](/learn/queueing-a-series-of-state-updates)。通常情况下，这是你想要的。然而，在这里它引起了一个问题，因为 `setTodos` 并不立即更新DOM。所以当你把列表滚动到最后一个元素时，该 todo 还没有被添加。这就是为什么滚动总是 “滞后” 一个 item 的原因。

为了解决这个问题，你可以强制React同步更新（ “flush” ）DOM。要做到这一点，从 `react-dom` 中导入 `flushSync` ，并将**状态更新包装**成一个 `flushSync` 回调：

```js
flushSync(() => {
  setTodos([ ...todos, newTodo]);
});
listRef.current.lastChild.scrollIntoView();
```

这将指示React在 `flushSync` 包装的代码执行后同步更新 DOM。因此，当你试图滚动到最后一个 todo 时，它已经在 DOM 中了：

<Sandpack>

```js
import { useState, useRef } from 'react';
import { flushSync } from 'react-dom';

export default function TodoList() {
  const listRef = useRef(null);
  const [text, setText] = useState('');
  const [todos, setTodos] = useState(
    initialTodos
  );

  function handleAdd() {
    const newTodo = { id: nextId++, text: text };
    flushSync(() => {
      setText('');
      setTodos([ ...todos, newTodo]);      
    });
    listRef.current.lastChild.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest'
    });
  }

  return (
    <>
      <button onClick={handleAdd}>
        Add
      </button>
      <input
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <ul ref={listRef}>
        {todos.map(todo => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
    </>
  );
}

let nextId = 0;
let initialTodos = [];
for (let i = 0; i < 20; i++) {
  initialTodos.push({
    id: nextId++,
    text: 'Todo #' + (i + 1)
  });
}
```

</Sandpack>

</DeepDive>

## 使用 ref 进行 DOM 操作的最佳实践 {/*best-practices-for-dom-manipulation-with-refs*/}

Refs are an escape hatch. You should only use them when you have to "step outside React". Common examples of this include managing focus, scroll position, or calling browser APIs that React does not expose.

If you stick to non-destructive actions like focusing and scrolling, you shouldn't encounter any problems. However, if you try to **modify** the DOM manually, you can risk conflicting with the changes React is making.

To illustrate this problem, this example includes a welcome message and two buttons. The first button toggles its presence using [conditional rendering](/learn/conditional-rendering) and [state](/learn/state-a-components-memory), as you would usually do in React. The second button uses the [`remove()` DOM API](https://developer.mozilla.org/en-US/docs/Web/API/Element/remove) to forcefully remove it from the DOM outside of React's control.

Try pressing "Toggle with setState" a few times. The message should disappear and appear again. Then press "Remove from the DOM". This will forcefully remove it. Finally, press "Toggle with setState":

<Sandpack>

```js
import {useState, useRef} from 'react';

export default function Counter() {
  const [show, setShow] = useState(true);
  const ref = useRef(null);

  return (
    <div>
      <button
        onClick={() => {
          setShow(!show);
        }}>
        Toggle with setState
      </button>
      <button
        onClick={() => {
          ref.current.remove();
        }}>
        Remove from the DOM
      </button>
      {show && <p ref={ref}>Hello world</p>}
    </div>
  );
}
```

```css
p,
button {
  display: block;
  margin: 10px;
}
```

</Sandpack>

After you've manually removed the DOM element, trying to use `setState` to show it again will lead to a crash. This is because you've changed the DOM, and React doesn't know how to continue managing it correctly.

**Avoid changing DOM nodes managed by React.** Modifying, adding children to, or removing children from elements that are managed by React can lead to inconsistent visual results or crashes like above.

However, this doesn't mean that you can't do it at all. It requires caution. **You can safely modify parts of the DOM that React has _no reason_ to update.** For example, if some `<div>` is always empty in the JSX, React won't have a reason to touch its children list. Therefore, it is safe to manually add or remove elements there.

<Recap>

- Refs are a generic concept, but most often you'll use them to hold DOM elements.
- You instruct React to put a DOM node into `myRef.current` by passing `<div ref={myRef}>`.
- Usually, you will use refs for non-destructive actions like focusing, scrolling, or measuring DOM elements.
- A component doesn't expose its DOM nodes by default. You can opt into exposing a DOM node by using `forwardRef` and passing the second `ref` argument down to a specific node.
- Avoid changing DOM nodes managed by React.
- If you do modify DOM nodes managed by React, modify parts that React has no reason to update.

</Recap>



<Challenges>

#### Play and pause the video {/*play-and-pause-the-video*/}

In this example, the button toggles a state variable to switch between a playing and a paused state. However, in order to actually play or pause the video, toggling state is not enough. You also need to call [`play()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/play) and [`pause()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/pause) on the DOM element for the `<video>`. Add a ref to it, and make the button work.

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function VideoPlayer() {
  const [isPlaying, setIsPlaying] = useState(false);

  function handleClick() {
    const nextIsPlaying = !isPlaying;
    setIsPlaying(nextIsPlaying);
  }

  return (
    <>
      <button onClick={handleClick}>
        {isPlaying ? 'Pause' : 'Play'}
      </button>
      <video width="250">
        <source
          src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
          type="video/mp4"
        />
      </video>
    </>
  )
}
```

```css
button { display: block; margin-bottom: 20px; }
```

</Sandpack>

For an extra challenge, keep the "Play" button in sync with whether the video is playing even if the user right-clicks the video and plays it using the built-in browser media controls. You might want to listen to `onPlay` and `onPause` on the video to do that.

<Solution>

Declare a ref and put it on the `<video>` element. Then call `ref.current.play()` and `ref.current.pause()` in the event handler depending on the next state.

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function VideoPlayer() {
  const [isPlaying, setIsPlaying] = useState(false);
  const ref = useRef(null);

  function handleClick() {
    const nextIsPlaying = !isPlaying;
    setIsPlaying(nextIsPlaying);

    if (nextIsPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  }

  return (
    <>
      <button onClick={handleClick}>
        {isPlaying ? 'Pause' : 'Play'}
      </button>
      <video
        width="250"
        ref={ref}
        onPlay={() => setIsPlaying(true)}
        onPause={() => setIsPlaying(false)}
      >
        <source
          src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
          type="video/mp4"
        />
      </video>
    </>
  )
}
```

```css
button { display: block; margin-bottom: 20px; }
```

</Sandpack>

In order to handle the built-in browser controls, you can add `onPlay` and `onPause` handlers to the `<video>` element and call `setIsPlaying` from them. This way, if the user plays the video using the browser controls, the state will adjust accordingly.

</Solution>

#### Focus the search field {/*focus-the-search-field*/}

Make it so that clicking the "Search" button puts focus into the field.

<Sandpack>

```js
export default function Page() {
  return (
    <>
      <nav>
        <button>Search</button>
      </nav>
      <input
        placeholder="Looking for something?"
      />
    </>
  );
}
```

```css
button { display: block; margin-bottom: 10px; }
```

</Sandpack>

<Solution>

Add a ref to the input, and call `focus()` on the DOM node to focus it:

<Sandpack>

```js
import { useRef } from 'react';

export default function Page() {
  const inputRef = useRef(null);
  return (
    <>
      <nav>
        <button onClick={() => {
          inputRef.current.focus();
        }}>
          Search
        </button>
      </nav>
      <input
        ref={inputRef}
        placeholder="Looking for something?"
      />
    </>
  );
}
```

```css
button { display: block; margin-bottom: 10px; }
```

</Sandpack>

</Solution>

#### Scrolling an image carousel {/*scrolling-an-image-carousel*/}

This image carousel has a "Next" button that switches the active image. Make the gallery scroll horizontally to the active image on click. You will want to call [`scrollIntoView()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView) on the DOM node of the active image:

```js
node.scrollIntoView({
  behavior: 'smooth',
  block: 'nearest',
  inline: 'center'
});
```

<Hint>

You don't need to have a ref to every image for this exercise. It should be enough to have a ref to the currently active image, or to the list itself. Use `flushSync` to ensure the DOM is updated *before* you scroll.

</Hint>

<Sandpack>

```js
import { useState } from 'react';

export default function CatFriends() {
  const [index, setIndex] = useState(0);
  return (
    <>
      <nav>
        <button onClick={() => {
          if (index < catList.length - 1) {
            setIndex(index + 1);
          } else {
            setIndex(0);
          }
        }}>
          Next
        </button>
      </nav>
      <div>
        <ul>
          {catList.map((cat, i) => (
            <li key={cat.id}>
              <img
                className={
                  index === i ?
                    'active' :
                    ''
                }
                src={cat.imageUrl}
                alt={'Cat #' + cat.id}
              />
            </li>
          ))}
        </ul>
      </div>
    </>
  );
}

const catList = [];
for (let i = 0; i < 10; i++) {
  catList.push({
    id: i,
    imageUrl: 'https://placekitten.com/250/200?image=' + i
  });
}

```

```css
div {
  width: 100%;
  overflow: hidden;
}

nav {
  text-align: center;
}

button {
  margin: .25rem;
}

ul,
li {
  list-style: none;
  white-space: nowrap;
}

li {
  display: inline;
  padding: 0.5rem;
}

img {
  padding: 10px;
  margin: -10px;
  transition: background 0.2s linear;
}

.active {
  background: rgba(0, 100, 150, 0.4);
}
```

</Sandpack>

<Solution>

You can declare a `selectedRef`, and then pass it conditionally only to the current image:

```js
<li ref={index === i ? selectedRef : null}>
```

When `index === i`, meaning that the image is the selected one, the `<li>` will receive the `selectedRef`. React will make sure that `selectedRef.current` always points at the correct DOM node.

Note that the `flushSync` call is necessary to force React to update the DOM before the scroll. Otherwise, `selectedRef.current` would always point at the previously selected item.

<Sandpack>

```js
import { useRef, useState } from 'react';
import { flushSync } from 'react-dom';

export default function CatFriends() {
  const selectedRef = useRef(null);
  const [index, setIndex] = useState(0);

  return (
    <>
      <nav>
        <button onClick={() => {
          flushSync(() => {
            if (index < catList.length - 1) {
              setIndex(index + 1);
            } else {
              setIndex(0);
            }
          });
          selectedRef.current.scrollIntoView({
            behavior: 'smooth',
            block: 'nearest',
            inline: 'center'
          });            
        }}>
          Next
        </button>
      </nav>
      <div>
        <ul>
          {catList.map((cat, i) => (
            <li
              key={cat.id}
              ref={index === i ?
                selectedRef :
                null
              }
            >
              <img
                className={
                  index === i ?
                    'active'
                    : ''
                }
                src={cat.imageUrl}
                alt={'Cat #' + cat.id}
              />
            </li>
          ))}
        </ul>
      </div>
    </>
  );
}

const catList = [];
for (let i = 0; i < 10; i++) {
  catList.push({
    id: i,
    imageUrl: 'https://placekitten.com/250/200?image=' + i
  });
}

```

```css
div {
  width: 100%;
  overflow: hidden;
}

nav {
  text-align: center;
}

button {
  margin: .25rem;
}

ul,
li {
  list-style: none;
  white-space: nowrap;
}

li {
  display: inline;
  padding: 0.5rem;
}

img {
  padding: 10px;
  margin: -10px;
  transition: background 0.2s linear;
}

.active {
  background: rgba(0, 100, 150, 0.4);
}
```

</Sandpack>

</Solution>

#### Focus the search field with separate components {/*focus-the-search-field-with-separate-components*/}

Make it so that clicking the "Search" button puts focus into the field. Note that each component is defined in a separate file and shouldn't be moved out of it. How do you connect them together?

<Hint>

You'll need `forwardRef` to opt into exposing a DOM node from your own component like `SearchInput`.

</Hint>

<Sandpack>

```js App.js
import SearchButton from './SearchButton.js';
import SearchInput from './SearchInput.js';

export default function Page() {
  return (
    <>
      <nav>
        <SearchButton />
      </nav>
      <SearchInput />
    </>
  );
}
```

```js SearchButton.js
export default function SearchButton() {
  return (
    <button>
      Search
    </button>
  );
}
```

```js SearchInput.js
export default function SearchInput() {
  return (
    <input
      placeholder="Looking for something?"
    />
  );
}
```

```css
button { display: block; margin-bottom: 10px; }
```

</Sandpack>

<Solution>

You'll need to add an `onClick` prop to the `SearchButton`, and make the `SearchButton` pass it down to the browser `<button>`. You'll also pass a ref down to `<SearchInput>`, which will forward it to the real `<input>` and populate it. Finally, in the click handler, you'll call `focus` on the DOM node stored inside that ref.

<Sandpack>

```js App.js
import { useRef } from 'react';
import SearchButton from './SearchButton.js';
import SearchInput from './SearchInput.js';

export default function Page() {
  const inputRef = useRef(null);
  return (
    <>
      <nav>
        <SearchButton onClick={() => {
          inputRef.current.focus();
        }} />
      </nav>
      <SearchInput ref={inputRef} />
    </>
  );
}
```

```js SearchButton.js
export default function SearchButton({ onClick }) {
  return (
    <button onClick={onClick}>
      Search
    </button>
  );
}
```

```js SearchInput.js
import { forwardRef } from 'react';

export default forwardRef(
  function SearchInput(props, ref) {
    return (
      <input
        ref={ref}
        placeholder="Looking for something?"
      />
    );
  }
);
```

```css
button { display: block; margin-bottom: 10px; }
```

</Sandpack>

</Solution>

</Challenges>
