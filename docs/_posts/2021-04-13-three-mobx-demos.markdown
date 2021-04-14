---
title:  "Two MobX Use Cases That Actually Worked Well"
date:   2021-04-13 07:00:00 +0200
categories: react
---

This week I decided to play with the well known and mostly loved MobX library. I worked on projects that used MobX before and was usually able to make the modifications I needed in the project, but until now I didn't spend the time to read its documentation and understand its use cases.

After reading I was able to come up with two main scenarios in which I liked the use of MobX. I hope you'll find this guided tour useful.

## What People Don't Like About React
Let's begin with the painful truth - React can still be rough around the edges. Although writing a basic component should work well with the first tutorial, you'll probably (or have probably already) hit a brick trying to do the following:

1. Synchronise global application-wide data.

2. Save an array or object as local state.

3. use useEffect anywhere in your components.

And don't get me wrong, React has some great solutions to each of these problems: We can use context to manage global state, we can use functional programming paradigms to store mutable data structures in state without actually mutating them, and we can write custom hooks to wrap the complexity of useEffect. However all of these solutions bind our business logic to React where no such binding is required.

Most importantly, React does not offer a standard way for business logic to communicate with our components.

MobX provides a good solution to the first two problems, i.e. it helped me store application-wide global state and share it between components, and to save deeply nested data structures as state variables.

## Managing Global State with a MobX Singleton
Our first challenge was to manage global application state. A global state, as opposed to local state, is stored in a singleton object, initialized when the application starts and is accessible from anywhere in the code. Any React component that uses the global state is updated as soon as the state changes. The state itself can change in response to events, such as information coming in from a web socket, or user interaction with the components.

Using mobx terminology we're creating a global singleton store which is being observed by the React components using it.

For our first demo I'll create an application to manage some global counters. Each counter has a value, and various components on screen access and modify the values.

Here's a codesandbox with the working code:

<iframe src="https://codesandbox.io/embed/stoic-butterfly-7n0r2?fontsize=14&hidenavigation=1&theme=dark"
style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
title="stoic-butterfly-7n0r2"
allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
></iframe>

The first interesting file is `logic/counters_store.jsx`. This file defines two classes and one singleton object, which is also being exported:

```
const counterStore = new CountersStore();
```

By exporting the result of new we created a singleton. Now every other file in the project that will import the default object from `counters_store` will get the same object.

A React component in any file in the project can now call:

```
import countersStore from "../logic/counters_store";
```

And it will have a reference to the same array of counters.

Of course for this we only needed JavaScript. The magic of MobX is called observable, and the idea is that this global state can "update", and when it updates it will automatically trigger a `setState` on each React component that used it.

The tricky part in MobX (compared with Redux for example), is that MobX will actually create its own getter methods on each global field that is observable, in order to know which fields should trigger a re-render. Examine this simple React component from the demo project:

```
export const MaxCounterValue = observer(function MaxCounterValue() {
    return (
        <div>
          <p>The max value is: {countersStore.maxValue}</p>
        </div>
        );
    });
```

In its render method it read only one property from countersStore: the value of `maxValue`. Thus, automatically, whenever `maxValue` changes MobX will trigger a new setState causing this component to re-render.

In order to make this magic we need to use two MobX mechanisms. The first, in the original class, is the function makeObservable. Here's the full source code for CountersStore class:

```
class CountersStore {
  constructor(count = 4) {
    this.counters = new Array(count).fill(0).map((c) => new Counter(c));
    makeObservable(this, {
      counters: observable,
      maxValue: computed
    });
  }

  get maxValue() {
    return this.counters.reduce((a, b) => (a.value > b.value ? a : b)).value;
  }
}
```

By defining counters as observable I tell MobX to notify listeners every time the array changes. The `computed` attribute given to `maxValue` causes MobX to listen to the (other) getters called in the function and trigger a re-evaluation when any of them changes.

The second mechanism is the Higher Order Component `observer` function in `mobx-react-lite` package. This Higher Order Component is responsible for re-rendering our component when observable data in MobX changes. When I wrote these demos and in real-life outside this blog, I would usually forget to call `observer` and my UI would not refresh.

## Managing Nested React State with a MobX Local Observable
A second cool use case for mobx is a replacement to functional programming paradigms when I need to store an object or array in state. In normal React code this snippet would not work as expected:

```
function Counters(props) {
  const { count } = props;
  const [counters, setCounters] = useState(() => new Array(count).fill(0));

  function click(index) {
    counters[index]++;
    setCounters(counters);
  }

  return (
    <div>
      {counters.map((v, i) => (
        <button onClick={() => click(i)}>{v}</button>
      ))}
    </div>
  );
}
```

Modifying the value in `counters` array did not change the memory address of `counters` array itself, and so setCounters call was actually a no-op.

A simple fix is to create a new `counters` array each time an item changes by replaing the `setCounters(counters)` call with:

```
setCounters([...counters]);
```

Which works but feels weird and might not be the most efficient. Another easy fix is to create yet another state variable just for re-rendering purposes.

With MobX we can create `counters` array as a MobX observable store, and get the same benefits as if it was a singleton global store. Here's the result in codesandbox:


<iframe src="https://codesandbox.io/embed/friendly-wind-m9rmq?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="friendly-wind-m9rmq"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

And the code itself:

```
const Counters = observer(function Counters(props) {
  const { count } = props;
  const counters = useLocalObservable(() => new Array(count).fill(0));

  function click(index) {
    counters[index]++;
  }

  return (
    <div>
      {counters.map((v, i) => (
        <button onClick={() => click(i)}>{v}</button>
      ))}
    </div>
  );
});
```

A local observable replaces immutable data libraries such as Immutable.JS or immer, and in my opinion results in cleaner code.

## fin
MobX is actually a very big library with many features, however when I used most of them it just made my code worse.

Notably MobX has a Reactions mechanism (the functions `autorun`, `reaction` and `when`) which seemed like a good idea but was actually worse than useEffect wrapped in custom hooks in any situation I found.

I also found M×obX's support of subclassing rather useless. Complex inheritance hierarchies and global state do not play nicely together.

After playing with MobX I can recommend these 3 business logic strategies:

1. Use small utility functions for state-less business logic.

2. Use custom hooks for stateful business logic that lives within the context of a component

3. Use MobX for global application-wide stateful business logic, or to store deeply nested data structures in state.

