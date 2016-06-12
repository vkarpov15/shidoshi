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

# Step 2: Multiple Components with react-redux

Now that you've seen the basics of how redux works, let's break ground on
the conduit project. Let's start with the global feed of posts. When you
open up the productionready.io site, you'll
see there's an HTTP request to this '/api/articles' endpoint that returns
a list of articles. Let's show this list in our code.

First, let's refactor out the App component into it's own file, add in
react-redux, and build out the UI that's going to contain the global
feed.

### Introducing react-redux

```javascript
import App from './components/App';
import { Provider } from 'react-redux';
import ReactDOM from 'react-dom';
import React from 'react';
import { applyMiddleware, createStore } from 'redux';

const defaultState = {
  appName: 'conduit',
  articles: null
};
const reducer = function(state = defaultState, action) {
  return state;
};

ReactDOM.render((
  <Provider store={store}>
    <App />
  </Provider>
), document.getElementById('main'));
```

The react-redux module is the "official bindings" between react and redux.
It adds some convenient syntactic sugar for binding your components to
your redux state. This "Provider" component that you get from react-redux
is how you tell react-redux about your redux store. Next, let's take a
look at the new App.js:

```javascript
import React from 'react';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  appName: state.appName
});

class App extends React.Component {
  render() {
    return (
      <div>
        {this.props.appName}
      </div>
    );
  }
}

export default connect(mapStateToProps, () => ({}))(App);
```

The `connect()` function is how react-redux binds your redux state to your
components. This `mapStateToProps` function maps the global redux state to
the specific pieces of state that your component cares about and puts them
into the 'props' property. This is mostly
for performance and testability purposes. The `connect()` function then
takes in your `mapStateToProps` function and ties it to your App component.

Don't worry about this second function for now, you'll learn about it later.

### Multiple Components

Now that you've got react-redux running, let's add some new components.
So far, all you have is this App component, which doesn't do much.

```javascript
import Header from './Header';
import Home from './Home';
import React from 'react';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  appName: state.appName
});

class App extends React.Component {
  render() {
    return (
      <div>
        <Header appName={this.props.appName} />
        <Home />
      </div>
    );
  }
}

export default connect(mapStateToProps, () => ({}))(App);
```

First up is this "Header" component. It's going to display the nav bar that
you see up at the top when you visit ProductionReady. Let's add in a
`Header.js` file.

```javascript
'use strict';

import React from 'react';

class Header extends React.Component {
  render() {
    return (
      <nav className="navbar navbar-light">
        <div className="container">

          <a className="navbar-brand">
            {this.props.appName.toLowerCase()}
          </a>
        </div>
      </nav>
    );
  }
}

export default Header;
```

Next, let's build up this 'Home' component. This component is going to
contain the global feed. The Home component is going to be pretty complicated,
so let's separate it out into it's own directory. The top-level component
is going to take in the top-level app name as a property from react-redux:

```javascript
import Banner from './Banner';
import MainView from './MainView';
import React from 'react';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  appName: state.appName
});

class Home extends React.Component {
  render() {
    return (
      <div className="home-page">

        <Banner appName={this.props.appName} />

        <div className="container page">
          <div className="row">
            <MainView />

            <div className="col-md-3">
              <div className="sidebar">

                <p>Popular Tags</p>

              </div>
            </div>
          </div>
        </div>

      </div>
    );
  }
}

export default connect(mapStateToProps, () => ({}))(Home);
```

This component is going to have 2 sub-components: Banner and MainView.
The Banner component is going to take in a property, the 'appName', which
the Home component gets from react-redux.
The Banner component is this big hero banner. The component's pretty
straightforward, just HTML and CSS.

```javascript
import React from 'react';

const Banner = ({ appName }) => {
  return (
    <div className="banner">
      <div className="container">
        <h1 className="logo-font">
          {appName.toLowerCase()}
        </h1>
        <p>A place to share your knowledge.</p>
      </div>
    </div>
  );
};

export default Banner;
```

Notice that, in this case, the component is a function, not a class.
This function shorthand lets you declare simple components more easily.

Next up, let's create the 'MainView' component. This component will get a
list of articles from the redux state and pass the list off to an "ArticleList"
component.

```javascript
import ArticleList from '../ArticleList';
import React from 'react';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  articles: state.articles
});

const MainView = props => {
  return (
    <div className="col-md-9">
      <div className="feed-toggle">
        <ul className="nav nav-pills outline-active">

        <li className="nav-item">
          <a
            href=""
            className="nav-link active">
            Global Feed
          </a>
        </li>

        </ul>
      </div>

      <ArticleList
        articles={props.articles} />
    </div>
  );
};

export default connect(mapStateToProps, () => ({}))(MainView);
```

The ArticleList component will be responsible for actually
rendering the list of articles, so let's implement that.

```javascript
import React from 'react';

const ArticleList = props => {
  if (!props.articles) {
    return (
      <div className="article-preview">Loading...</div>
    );
  }

  if (props.articles.length === 0) {
    return (
      <div className="article-preview">
        No articles are here... yet.
      </div>
    );
  }

  return (
    <div>
      {
        props.articles.map(article => {
          return (
            <h2>{article.title}</h2>
          );
        })
      }
    </div>
  );
};

export default ArticleList;
```

If articles is undefined, we'll assume the articles are loading. If
there are no articles, we'll put a preview message, and otherwise we'll
display an H2 with the article title.

Great! So now when you pull up your local server you'll see this
"Loading..." message. Next step is to actually run the HTTP request and
put the result into the redux state.

---------------------------

# Step 3: Displaying the Global Feed with Middleware

First step, let's add an `agent.js` file that's going to use the superagent
HTTP client library to make an HTTP request to the ProductionReady API.

```javascript
'use strict';

import superagentPromise from 'superagent-promise';
import _superagent from 'superagent';

const superagent = superagentPromise(_superagent, global.Promise);

const API_ROOT = 'https://conduit.productionready.io/api';

const responseBody = res => res.body;

const requests = {
  get: url =>
    superagent.get(`${API_ROOT}${url}`).then(responseBody)
};

const Articles = {
  all: page =>
    requests.get(`/articles?limit=10`)
};

export default {
  Articles
};
```

All you need to do now is get the value that the `agent.Articles.all()`
promise resolves to into redux. Let's call this function when the `Home`
component loads.

### Promise middleware

```javascript
import Banner from './Banner';
import MainView from './MainView';
import React from 'react';
import agent from '../../agent';
import { connect } from 'react-redux';

const Promise = global.Promise;

const mapStateToProps = state => ({
  appName: state.appName
});

const mapDispatchToProps = dispatch => ({
  onLoad: (payload) =>
    dispatch({ type: 'HOME_PAGE_LOADED', payload }),
});

class Home extends React.Component {
  componentWillMount() {
    this.props.onLoad(agent.Articles.all());
  }

  render() {
    // ...
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(Home);
```

This `mapDispatchToProps` function is the second function that gets passed
to react-redux's `connect()` function. The `mapDispatchToProps()` function
maps the Redux store's `dispatch()` function to functions. Each function that
`mapDispatchToProps()` returns gets attached to the component's `props`. This
means your component can call `this.props.onLoad()` to fire off an
event with type 'HOME_PAGE_LOADED' and a 'payload'. This payload will contain
the HTTP promise.

The right place to call the promise is in the `componentWillMount()` function.
This `componentWillMount()` function is what's known as a "lifecycle hook"
It'll get called when this component gets created, so let's put the
initialization logic here.

Great, so now you're dispatching an action that has a 'payload' property that
contains a promise. Redux has a concept called "middleware", which lets you
intercept and transform actions. Let's write some middleware that'll intercept
actions where the 'payload' property is a promise.

```javascript
const promiseMiddleware = store => next => action => {
  if (isPromise(action.payload)) {
    action.payload.then(
      res => {
        action.payload = res;
        store.dispatch(action);
      },
      error => {
        action.error = true;
        action.payload = error.response.body;
        store.dispatch(action);
      }
    );

    return;
  }

  next(action);
};

function isPromise(v) {
  return v && typeof v.then === 'function';
}

export {
  promiseMiddleware
};
```

Great, now you need to plug this middleware into the redux store. Back in
`index.js`, let's use this `createMiddleware` function that redux exports to
plug the promise middleware into redux.

```javascript
import App from './components/App';
import { Provider } from 'react-redux';
import ReactDOM from 'react-dom';
import React from 'react';
import { applyMiddleware, createStore } from 'redux';
import { promiseMiddleware } from './middleware';

const defaultState = {
  // ...
};
const reducer = function(state = defaultState, action) {
  // ...
};
const store = createStore(reducer, applyMiddleware(promiseMiddleware));

// ...
```

### Article Preview

With this code you now get an ugly list that contains the list of
articles. Let's make this prettier and add in yet another component.
Let's call this component "ArticlePreview". It'll take in an article
and display the standard ProductionReady preview for it.

```javascript
import React from 'react';

const ArticlePreview = props => {
  const article = props.article;

  return (
    <div className="article-preview">
      <div className="article-meta">
        <a>
          <img src={article.author.image} />
        </a>

        <div className="info">
          <a className="author">
            {article.author.username}
          </a>
          <span className="date">
            {new Date(article.createdAt).toDateString()}
          </span>
        </div>

        <div className="pull-xs-right">
          <button
            className="btn btn-sm btn-outline-primary">
            <i className="ion-heart"></i> {article.favoritesCount}
          </button>
        </div>
      </div>

      <a to={`article/${article.slug}`} className="preview-link">
        <h1>{article.title}</h1>
        <p>{article.description}</p>
        <span>Read more...</span>
        <ul className="tag-list">
          {
            article.tagList.map(tag => {
              return (
                <li className="tag-default tag-pill tag-outline" key={tag}>
                  {tag}
                </li>
              )
            })
          }
        </ul>
      </a>
    </div>
  );
}

export default ArticlePreview;
```

Now, let's plug this component into the `ArticleList` component:

```javascript
import ArticlePreview from './ArticlePreview';
import React from 'react';

const ArticleList = props => {
  // ...
  return (
    <div>
      {
        props.articles.map(article => {
          return (
            <ArticlePreview article={article} />
          );
        })
      }
    </div>
  );
};
// ...
```

And now you have a pretty list of articles that looks just like
ProductionReady.

---------------

# Step 4: Multiple Views with React-Router

---------------

# Step 5: Authentication
