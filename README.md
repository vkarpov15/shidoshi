# Step 1: Toggle a Checkbox With Redux

The fundamental concepts in redux are called "stores" and "actions".
A store has two parts: a plain-old JavaScript object that represents the
current state of your application, and a "reducer", which is a function that
describes how incoming actions modify your state.

To create a store that represents the state of a checkbox, let's create a
default state, a reducer that allows actions of type 'TOGGLE' to change
the state, and finally use redux's `createStore()` function to create a new
store.

```javascript
const defaultState = { checked: false };
const reducer = function(state = defaultState, action) {
  switch (action.type) {
    case 'TOGGLE':
      return { ...state, checked: !state.checked };
  }
  return state;
};
const store = createStore(reducer);
```

Note that the reducer receives both the current state and the action as
parameters, and returns the modified state. A well-written reducer should
not have any side-effects, that is, if you call a reducer with the same state
and the same action, you should always get the same result. Reducers should
not modify or rely on any global state. It's also good practice to encapsulate
as much of your application logic as possible in reducers, because, since your
reducers don't rely on side-effects or global state, they're really easy to
test, debug, and refactor.

### Displaying the State


So great, now that you have a store, what do you do with it? A store has
three functions that you're going to be using in this course: `subscribe()`,
`dispatch()`, and `getState()`. `getState()` is simple, it just gets the
current state of the store. `dispatch()` is the function you use to fire
off a new action. `subscribe()` fires a callback every time a
new action is fired **after** your reducer is done, so if you call `getState()`
in the `subscribe()` callback you get the newly reduced state.

First off, let's make the App component subscribe to the state. The right
place to do that is in the `componentWillMount()` function. This function
is an example of what's known as a React component "life-cycle hook".
The `componentWillMount()` function is important here because it gets called
when the component is going to be created, so it's a good place to put
initialization logic.

Let's subscribe to the redux store and call React's `setState()` function
every time the store changes. React components also have their own internal
state. React calls the `render()` function every time the
component's state changes, and it's the `render()` function's job to
tell React how to draw the component.

```javascript
class App extends React.Component {
  constructor() {
    super();
    this.state = {};
  }

  componentWillMount() {
    store.subscribe(() => this.setState(store.getState()));
  }
}
```

Let's tweak the App component to display a checkbox whose state depends on
the redux store. When the `checked` state is falsy, note that the checkbox
is off. When you change `checked` to true by default, the checkbox is on.

```javascript
render() {
  return (
    <div>
      <h1>To-dos</h1>
      <div>
        Learn Redux&nbsp;
        <input
          type="checkbox"
          checked={!!this.state.checked} />
      </div>
      {
        this.state.checked ? (<h2>Done!</h2>) : null
      }
    </div>
  );
}
```

What happens when you click on the checkbox? Absolutely nothing! What gives?
One of the key ideas of React is that what gets displayed is a pure function
of the component's state. In other words, since your state doesn't change,
React will always render the checkbox as checked.

### Dispatching Actions

In order to mutate the redux state, you need to dispatch an action. Recall
that an action is the 2nd parameter that gets passed to your reducer. An
action is a plain-old JavaScript object that tells your reducer what happened.
Let's create a function `onClick` that calls the redux store's `dispatch()`
function, which is how you fire off a new action in redux.

```javascript
render() {
  const onClick = () => store.dispatch({ type: 'TOGGLE' });
  return (
    <div>
      <h1>To-dos</h1>
      <div>
        Learn Redux&nbsp;
        <input
          type="checkbox"
          checked={!!this.state.checked}
          onClick={onClick} />
      </div>
      {
        this.state.checked ? (<h2>Done!</h2>) : null
      }
    </div>
  );
}
```

Redux actions
**must** have a `type` property, so let's create an action with type `TOGGLE`
to match the reducer, and add the `onClick` function as an `onClick` handler
for the checkbox. Now, if you look in the browser, you'll notice you can
actually toggle the checkbox.

---------------
