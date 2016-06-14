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

So far you've scaffolded out a single view that displays the global feed.
It's a good start, but this site's going to need several more views, so
let's plug in react-router to make it happen.

### Refactor Store

First off, `index.js` is getting bloated, so let's refactor the redux store
out into a `store.js` file.

```javascript
import { applyMiddleware, createStore } from 'redux';
import { promiseMiddleware } from './middleware';

const defaultState = {
  appName: 'conduit',
  articles: null
};
const reducer = function(state = defaultState, action) {
  switch (action.type) {
    case 'HOME_PAGE_LOADED':
      return { ...state, articles: action.payload.articles };
  }
  return state;
};

const middleware = applyMiddleware(promiseMiddleware);

const store = createStore(reducer, middleware);

export default store;
```

And import this file in `index.js`. Now `index.js` is nice and clean,
and ready to handle routing.

### Setting Up The Routing Structure

React router uses components to map URLs to views. To use react-router,
import "Router", "Route", "IndexRoute", and "hashHistory" from react-router,
and declare the routing hierarchy in the `ReactDOM.render()` call.
The "Router" component instantiates the router, and then "Route" creates
a route for URLs that says URLs that start with "/" render the "App" component.
Routes can be nested, so let's create two nested routes.

The first route is an "IndexRoute" that displays the "Home" component. This
means that going to 'localhost:4000' will display the 'App' component with
the 'Home' component underneath. The second component will be the 'Login'
component, which will display the login form, so when you navigate to
'/login', you'll see the 'App' component with the 'Login' component
underneath.

```javascript
import { Provider } from 'react-redux';
import ReactDOM from 'react-dom';
import React from 'react';
import { Router, Route, IndexRoute, hashHistory } from 'react-router';

import App from './components/App';
import Home from './components/Home';
import Login from './components/Login';
import store from './store';

ReactDOM.render((
  <Provider store={store}>
    <Router history={hashHistory}>
      <Route path="/" component={App}>
        <IndexRoute component={Home} />
        <Route path="login" component={Login} />
      </Route>
    </Router>
  </Provider>
), document.getElementById('main'));
```

Now that your routing structure says which component should be rendered
within the 'App' component, the 'App' component can no longer hard-code the
'Home' component, so let's get rid of it. The component to be rendered is
represented by the `props.children` property if you add this magic
`App.contextTypes` snippet, which tells react-router to attach the `children`
property to this component's props.

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
        {this.props.children}
      </div>
    );
  }
}

App.contextTypes = {
  router: React.PropTypes.object.isRequired
};

export default connect(mapStateToProps, () => ({}))(App);
```

### The Login Component

Now that the routing is in place, let's scaffold out the 'Login' component.
For now, this component will just contain HTML, so there's no need for a
`mapStateToProps()` or `mapDispatchToProps()` function. Specifically,
this component will just contain a form with an input for the user to
enter their email and password.

```javascript
import React from 'react';
import { connect } from 'react-redux';

class Login extends React.Component {
  render() {
    return (
      <div className="auth-page">
        <div className="container page">
          <div className="row">

            <div className="col-md-6 offset-md-3 col-xs-12">
              <h1 className="text-xs-center">Sign In</h1>
              <p className="text-xs-center">
                <a>
                  Need an account?
                </a>
              </p>

              <form>
                <fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control form-control-lg"
                      type="email"
                      placeholder="Email" />
                  </fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control form-control-lg"
                      type="password"
                      placeholder="Password" />
                  </fieldset>

                  <button
                    className="btn btn-lg btn-primary pull-xs-right"
                    type="submit">
                    Sign in
                  </button>

                </fieldset>
              </form>
            </div>

          </div>
        </div>
      </div>
    );
  }
}

export default connect(() => ({}), () => ({}))(Login);
```

### React-Router Links

The login component is now wired up to appear when the user navigates to
'/login'. Let's now use the react-router `Link` component to create some
links to navigate back and forth between the two views. You shouldn't use
plain-old HTML 'a' tags with react-router, you need to use the `Link`
component.

```javascript
'use strict';

import { Link } from 'react-router';
import React from 'react';

class Header extends React.Component {
  render() {
    return (
      <nav className="navbar navbar-light">
        <div className="container">

          <Link to="/" className="navbar-brand">
            {this.props.appName.toLowerCase()}
          </Link>

          <ul className="nav navbar-nav pull-xs-right">
            <li className="nav-item">
              <Link to="/" className="nav-link">
                Home
              </Link>
            </li>

            <li className="nav-item">
              <Link to="login" className="nav-link">
                Sign in
              </Link>
            </li>
          </ul>
        </div>
      </nav>
    );
  }
}

export default Header;
```

And now you can switch back and forth between the sign-in and home views.

---------------

# Step 5: Authentication

It's time to actually hook up the login view. But first, in order to make
this easier, let's do some refactoring to break up the reducers into smaller
independent chunks.

### Refactoring to use `combineReducers()`

You can write one big monolithic reducer, but that makes tasks like cleaning
up state when the view changes confusing. Redux's `combineReducers()` function
lets you build up a reducer out of independent reducers. First, let's write
3 separate reducers. First, let's take the global feed reducer logic from
the home page and stuff it into 'reducers/home.js':

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'HOME_PAGE_LOADED':
      return {
        ...state,
        articles: action.payload.articles
      };
  }

  return state;
};
```

Next, let's create a 'common' reducer that'll take care of the `appName`
property:

```javascript
const defaultState = {
  appName: 'Conduit'
};

export default (state = defaultState, action) => {
  return state;
};
```

Finally, let's create a `reducers/auth.js` file that will contain all the
auth-specific logic. For now, it'll do nothing.

```javascript
export default (state = {}, action) => {
  return state;
};
```

In `store.js`, let's import in these reducers and build them into a single
reducer using `combineReducers()`.

```javascript
import { applyMiddleware, createStore, combineReducers } from 'redux';
import { promiseMiddleware } from './middleware';
import auth from './reducers/auth';
import common from './reducers/common';
import home from './reducers/home';

const reducer = combineReducers({
  auth,
  common,
  home
});

const middleware = applyMiddleware(promiseMiddleware);

const store = createStore(reducer, middleware);

export default store;
```

With these changes, we're going to need to change the `mapStateToProps`
function for all components that have them. For example, in `App.js`,
remember that `appName` is now in the `common` reducer? To account for that,
`mapDispatchToProps()` needs to get `appName` from the state's `common`
property.

```javascript
import Header from './Header';
import Home from './Home';
import React from 'react';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  appName: state.common.appName
});

class App extends React.Component {
  // ...
}

App.contextTypes = {
  router: React.PropTypes.object.isRequired
};

export default connect(mapStateToProps, () => ({}))(App);
```

Now we need to apply the same changes to the `Home` component:

```javascript
import Banner from './Banner';
import MainView from './MainView';
import React from 'react';
import agent from '../../agent';
import { connect } from 'react-redux';

const Promise = global.Promise;

const mapStateToProps = state => ({
  appName: state.common.appName
});

const mapDispatchToProps = dispatch => ({
  // ...
});

class Home extends React.Component {
  // ...
}

export default connect(mapStateToProps, mapDispatchToProps)(Home);
```

And the `MainView` component:

```javascript
import ArticleList from '../ArticleList';
import React from 'react';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  articles: state.home.articles
});

const MainView = props => {
  // ...
};

export default connect(mapStateToProps, () => ({}))(MainView);
```

### Wiring the Login Component

First, let's add a `login()` function to `agent.js` that'll actually fire
off the HTTP request.

```javascript
// ...

const requests = {
  get: url =>
    superagent.get(`${API_ROOT}${url}`).then(responseBody),
  post: (url, body) =>
    superagent.post(`${API_ROOT}${url}`, body).then(responseBody)
};

const Articles = {
  // ...
};

const Auth = {
  login: (email, password) =>
    requests.post('/users/login', { user: { email, password } })
};

export default {
  Articles,
  Auth
};
```

So now, the login component needs to call `agent.Auth.login()` with the
username and password the user specifies. The Login component and auth
reducer now need to be able to get the user's input and call the `login()`
function. First, in the 'Login' component, we need to pull in all the state
from the 'auth' reducer in `mapStateToProps()` and a `mapDispatchToProps()`
function that's going to dispatch separate events:

```javascript
import { Link } from 'react-router';
import ListErrors from './ListErrors';
import React from 'react';
import agent from '../agent';
import { connect } from 'react-redux';

const mapStateToProps = state => ({ ...state.auth });

const mapDispatchToProps = dispatch => ({
  onChangeEmail: value =>
    dispatch({ type: 'UPDATE_FIELD_AUTH', key: 'email', value }),
  onChangePassword: value =>
    dispatch({ type: 'UPDATE_FIELD_AUTH', key: 'password', value }),
  onSubmit: (email, password) =>
    dispatch({ type: 'LOGIN', payload: agent.Auth.login(email, password) })
});

class Login extends React.Component {
  constructor() {
    super();
    this.changeEmail = ev => this.props.onChangeEmail(ev.target.value);
    this.changePassword = ev => this.props.onChangePassword(ev.target.value);
    this.submitForm = (email, password) => ev => {
      ev.preventDefault();
      this.props.onSubmit(email, password);
    };
  }

  render() {
    const email = this.props.email;
    const password = this.props.password;
    return (
      <div className="auth-page">
        <div className="container page">
          <div className="row">

            <div className="col-md-6 offset-md-3 col-xs-12">
              <h1 className="text-xs-center">Sign In</h1>
              <p className="text-xs-center">
                <Link to="register">
                  Need an account?
                </Link>
              </p>

              <ListErrors errors={this.props.errors} />

              <form onSubmit={this.submitForm(email, password)}>
                <fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control form-control-lg"
                      type="email"
                      placeholder="Email"
                      value={email}
                      onChange={this.changeEmail} />
                  </fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control form-control-lg"
                      type="password"
                      placeholder="Password"
                      value={password}
                      onChange={this.changePassword} />
                  </fieldset>

                  <button
                    className="btn btn-lg btn-primary pull-xs-right"
                    type="submit"
                    disabled={this.props.inProgress}>
                    Sign in
                  </button>

                </fieldset>
              </form>
            </div>

          </div>
        </div>
      </div>
    );
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(Login);
```

There's also a ListErrors component that's going to display any errors
that occurred during login, like if the user entered the wrong password.
Let's implement this component:

```javascript
import React from 'react';

class ListErrors extends React.Component {
  render() {
    const errors = this.props.errors;
    if (errors) {
      return (
        <ul className="error-messages">
          {
            Object.keys(errors).map(key => {
              return (
                <li key={key}>
                  {key} {errors[key]}
                </li>
              );
            })
          }
        </ul>
      );
    } else {
      return null;
    }
  }
}

export default ListErrors;

```

The login component is going to dispatch 2 different events: an
'UPDATE_FIELD_AUTH' event that's going to fire when the user changes an
input field, and a 'LOGIN' event that's going to fire when the user submits
the login form.

The input fields need to take the current value of email and password, and
call the correct functions when they change. Also, if the state says there's
an auth request in progress, we'll disable the submit button.

### Wiring the Reducers

Next up is the auth reducer. Remember there's 2 events we need to handle
from the login component, 'LOGIN' and 'UPDATE_FIELD_AUTH', plus we need
to set the 'inProgress' field when there's a request in progress.
Let's write separate handlers for each of these events.

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'LOGIN':
      return {
        ...state,
        inProgress: false,
        errors: action.error ? action.payload.errors : null
      };
    case 'ASYNC_START':
      if (action.subtype === 'LOGIN' || action.subtype === 'REGISTER') {
        return { ...state, inProgress: true };
      }
      break;
    case 'UPDATE_FIELD_AUTH':
      return { ...state, [action.key]: action.value };
  }

  return state;
};
```

Let's also change the promise middleware to dispatch an 'ASYNC_START' action
when an async action starts:

```javascript
const promiseMiddleware = store => next => action => {
  if (isPromise(action.payload)) {
    store.dispatch({ type: 'ASYNC_START', subtype: action.type });
    action.payload.then(
      // ...
    );

    return;
  }

  next(action);
};

// ...
```

Great, now you can actually submit a login request and, if the password
is incorrect, you get an error. However, when you enter in the right password,
nothing happens.

### Local Storage Middleware and Redirects

Notice that when you make a successful login request, you get back a user
object that has a "token". You need to store this token, along with the
whole user object, in your redux state, and redirect the user back to the
home page. The right place to put this is
in your common reducer:

```javascript
const defaultState = {
  appName: 'Conduit',
  token: null
};

export default (state = defaultState, action) => {
  case 'APP_LOAD':
    return {
      ...state,
      token: action.token || null,
      appLoaded: true,
      currentUser: action.payload ? action.payload.user : null
    };
  case 'REDIRECT':
    return { ...state, redirectTo: null };
  case 'LOGIN':
    return {
      ...state,
      redirectTo: action.error ? null : '/',
      token: action.error ? null : action.payload.user.token,
      currentUser: action.error ? null : action.payload.user
    };
  return state;
};
```

We're going to take the user token and the user object from the action
and put it into the `common` property in our store's state. When the app
loads, we're going to take the currently logged-in user. Also, this
`redirectTo` property will say where the user should be redirected to.
Let's hook up the `redirectTo` property to the main `App` component,
because the `App` component already has access to the router.

```javascript
import Header from './Header';
import Home from './Home';
import React from 'react';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  appName: state.common.appName,
  redirectTo: state.common.redirectTo
});

const mapDispatchToProps = dispatch => ({
  onRedirect: () =>
    dispatch({ type: 'REDIRECT' })
});

class App extends React.Component {
  componentWillReceiveProps(nextProps) {
    if (nextProps.redirectTo) {
      this.context.router.replace(nextProps.redirectTo);
      this.props.onRedirect();
    }
  }

  render() {
    // ...
  }
}

App.contextTypes = {
  router: React.PropTypes.object.isRequired
};

export default connect(mapStateToProps, mapDispatchToProps)(App);
```

That's great, but in order to persist the token if the user closes the
window, you're going to have to put the token into `localStorage`. Let's
add a middleware to do this.

```javascript
const localStorageMiddleware = store => next => action => {
  if (action.type === 'REGISTER' || action.type === 'LOGIN') {
    if (!action.error) {
      window.localStorage.setItem('jwt', action.payload.user.token);
      agent.setToken(action.payload.user.token);
    }
  } else if (action.type === 'LOGOUT') {
    window.localStorage.setItem('jwt', '');
    agent.setToken(null);
  }

  next(action);
};

export {
  localStorageMiddleware,
  promiseMiddleware
};
```

Once you persist the token in local storage, you need to be able to pull
the token from local storage as well. Let's put that functionality in the
'App' component:

```javascript
import Header from './Header';
import Home from './Home';
import React from 'react';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  appName: state.common.appName,
  currentUser: state.common.currentUser,
  redirectTo: state.common.redirectTo
});

const mapDispatchToProps = dispatch => ({
  onLoad: (payload, token) =>
    dispatch({ type: 'APP_LOAD', payload, token }),
  onRedirect: () =>
    dispatch({ type: 'REDIRECT' })
});

class App extends React.Component {
  componentWillMount() {
    const token = window.localStorage.getItem('jwt');
    if (token) {
      agent.setToken(token);
    }

    this.props.onLoad(token ? agent.Auth.current() : null, token);
  }

  componentWillReceiveProps(nextProps) {
    if (nextProps.redirectTo) {
      this.context.router.replace(nextProps.redirectTo);
      this.props.onRedirect();
    }
  }

  render() {
    // ...
  }
}

App.contextTypes = {
  router: React.PropTypes.object.isRequired
};

export default connect(mapStateToProps, mapDispatchToProps)(App);
```

And add this `agent.Auth.current()` function, which is going to get
the currently logged-in user.

```javascript
const Auth = {
  current: () =>
    requests.get('/user'),
  login: (email, password) =>
    requests.post('/users/login', { user: { email, password } })
};

// ...

export default {
  Articles,
  Auth,
  setToken: _token => { token = _token; }
};
```

### Displaying Login Status in the Nav Bar

Finally, let's tweak the header to show which user is currently logged in.
We'll add 2 new components to the 'Header' component: one that shows up
when there's a logged-in user, and one that shows up when there isn't.

```javascript
'use strict';

import { Link } from 'react-router';
import React from 'react';

const LoggedOutView = props => {
  if (!props.currentUser) {
    return (
      <ul className="nav navbar-nav pull-xs-right">

        <li className="nav-item">
          <Link to="/" className="nav-link">
            Home
          </Link>
        </li>

        <li className="nav-item">
          <Link to="login" className="nav-link">
            Sign in
          </Link>
        </li>

        <li className="nav-item">
          <Link to="register" className="nav-link">
            Sign up
          </Link>
        </li>

      </ul>
    );
  }
  return null;
};

const LoggedInView = props => {
  if (props.currentUser) {
    return (
      <ul className="nav navbar-nav pull-xs-right">

        <li className="nav-item">
          <Link to="" className="nav-link">
            Home
          </Link>
        </li>

        <li className="nav-item">
          <Link to="editor" className="nav-link">
            <i className="ion-compose"></i>&nbsp;New Post
          </Link>
        </li>

        <li className="nav-item">
          <Link to="settings" className="nav-link">
            <i className="ion-gear-a"></i>&nbsp;Settings
          </Link>
        </li>

        <li className="nav-item">
          <Link
            to={`@${props.currentUser.username}`}
            className="nav-link">
            <img src={props.currentUser.image} className="user-pic" />
            {props.currentUser.username}
          </Link>
        </li>

      </ul>
    );
  }

  return null;
};

class Header extends React.Component {
  render() {
    return (
      <nav className="navbar navbar-light">
        <div className="container">

          <Link to="/" className="navbar-brand">
            {this.props.appName.toLowerCase()}
          </Link>

          <LoggedOutView currentUser={this.props.currentUser} />

          <LoggedInView currentUser={this.props.currentUser} />
        </div>
      </nav>
    );
  }
}

export default Header;
```

In order to get this to work, we're going to have to pass the 'currentUser'
property to the `Header` component from the `App` component, so let's
tweak the `App` component:

```javascript
import Header from './Header';
import Home from './Home';
import React from 'react';
import agent from '../agent';
import { connect } from 'react-redux';

// ...

class App extends React.Component {
  // ...

  render() {
    return (
      <div>
        <Header
          currentUser={this.props.currentUser}
          appName={this.props.appName} />
        {this.props.children}
      </div>
    );
  }
}

// ...
```

And now, when you refresh, you should see the logged in view in the
nav bar.

---------------

# Step 6: Settings and Registration Forms

Next up is the registration and settings views. The settings view lets you
modify your profile, and the registration view is
a form that lets you sign up for a new account. Let's start off with the
registration view, because it's similar to the login view.

### Registration View

First off, let's add a `register` function to `agent.js`:

`src/reducers/agent.js`

```javascript
// ...

const Auth = {
  current: () =>
    requests.get('/user'),
  login: (email, password) =>
    requests.post('/users/login', { user: { email, password } }),
  register: (username, email, password) =>
    requests.post('/users', { user: { username, email, password } })
};

// ...
```

Now, we need a `Register` component and corresponding reducers. For the
reducers, first we need to update the auth state in the same way that
the login reducer does:

`src/reducers/auth.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'LOGIN':
    case 'REGISTER':
      return {
        ...state,
        inProgress: false,
        errors: action.error ? action.payload.errors : null
      };
  // ...
};
```

That handles registration errors. When registration succeeds, the
ProductionReady API gives you a token in the same way login does, so
we also need to change the common reducer:

`src/reducers/common.js`

```javascript
export default (state = defaultState, action) => {
  switch (action.type) {
    // ...
    case 'LOGIN':
    case 'REGISTER':
      return {
        ...state,
        redirectTo: action.error ? null : '/',
        token: action.error ? null : action.payload.user.token,
        currentUser: action.error ? null : action.payload.user
      };
  }
  return state;
};
```

Like login, when register succeeds we'll redirect to the main view and
attach the token and current user from the payload. Now, let's add the
Register component.

`src/components/Register.js`

```javascript
import { Link } from 'react-router';
import ListErrors from './ListErrors';
import React from 'react';
import agent from '../agent';
import { connect } from 'react-redux';

const mapStateToProps = state => ({ ...state.auth });

const mapDispatchToProps = dispatch => ({
  onChangeEmail: value =>
    dispatch({ type: 'UPDATE_FIELD_AUTH', key: 'email', value }),
  onChangePassword: value =>
    dispatch({ type: 'UPDATE_FIELD_AUTH', key: 'password', value }),
  onChangeUsername: value =>
    dispatch({ type: 'UPDATE_FIELD_AUTH', key: 'username', value }),
  onSubmit: (username, email, password) => {
    const payload = agent.Auth.register(username, email, password);
    dispatch({ type: 'REGISTER', payload })
  }
});

class Register extends React.Component {
  constructor() {
    super();
    this.changeEmail = ev => this.props.onChangeEmail(ev.target.value);
    this.changePassword = ev => this.props.onChangePassword(ev.target.value);
    this.changeUsername = ev => this.props.onChangeUsername(ev.target.value);
    this.submitForm = (username, email, password) => ev => {
      ev.preventDefault();
      this.props.onSubmit(username, email, password);
    }
  }

  render() {
    const email = this.props.email;
    const password = this.props.password;
    const username = this.props.username;

    return (
      <div className="auth-page">
        <div className="container page">
          <div className="row">

            <div className="col-md-6 offset-md-3 col-xs-12">
              <h1 className="text-xs-center">Sign Up</h1>
              <p className="text-xs-center">
                <Link to="login">
                  Have an account?
                </Link>
              </p>

              <ListErrors errors={this.props.errors} />

              <form onSubmit={this.submitForm(username, email, password)}>
                <fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control form-control-lg"
                      type="text"
                      placeholder="Username"
                      value={this.props.username}
                      onChange={this.changeUsername} />
                  </fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control form-control-lg"
                      type="email"
                      placeholder="Email"
                      value={this.props.email}
                      onChange={this.changeEmail} />
                  </fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control form-control-lg"
                      type="password"
                      placeholder="Password"
                      value={this.props.password}
                      onChange={this.changePassword} />
                  </fieldset>

                  <button
                    className="btn btn-lg btn-primary pull-xs-right"
                    type="submit"
                    disabled={this.props.inProgress}>
                    Sign in
                  </button>

                </fieldset>
              </form>
            </div>

          </div>
        </div>
      </div>
    );
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(Register);
```

Now, let's see the registration view in action. If you go to the `register`
route, you'll see this registration route, and you can register for a new
account and see yourself logged in.

### Settings

Now that you're done with the registration form, let's implement a more
complex form: the `Settings` form. The settings form lets you modify
your profile picture, your username, your bio, your email address, and
your password, as well as logout.

First off, let's add the ability to save the current user to `agent.js`.
For ProductionReady, this takes the form of a `put` request.

`src/agent.js`

```javascript
// ...

const Auth = {
  current: () =>
    requests.get('/user'),
  login: (email, password) =>
    requests.post('/users/login', { user: { email, password } }),
  register: (username, email, password) =>
    requests.post('/users', { user: { username, email, password } }),
  save: user =>
    requests.put('/user', { user })
};

// ...
```

Next, let's add a reducer to handle the settings view. In this case,
you're not going to store the state of inputs in the redux store, so
there's not going to be an 'UPDATE_FIELD' action like in the login and
registration views.

`src/reducers/settings.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'SETTINGS_SAVED':
      return {
        ...state,
        inProgress: false,
        errors: action.error ? action.payload.errors : null
      };
    case 'ASYNC_START':
      return {
        ...state,
        inProgress: true
      };
  }

  return state;
};
```

There's only 2 actions that this reducer will care about: 'ASYNC_START',
which  will fire when the user hits submit, and 'SETTINGS_SAVED', which will
fire when the save request to the server completes. Everything else will be
stored in the component itself. Let's tweak `store.js` to add this new reducer
to our main reducer.

`src/store.js`

```javascript
const reducer = combineReducers({
  auth,
  common,
  home,
  settings
});
```

Next up, let's add handlers for the logout button and the 'SETTINGS_SAVED'
button to the common reducer.

`src/reducers/common.js`

```javascript
// ...

export default (state = defaultState, action) => {
  switch (action.type) {
    // ...
    case 'REDIRECT':
      return { ...state, redirectTo: null };
    case 'LOGOUT':
      return { ...state, redirectTo: '/', token: null, currentUser: null };
    case 'SETTINGS_SAVED':
      return {
        ...state,
        redirectTo: action.error ? null : '/',
        currentUser: action.error ? null : action.payload.user
      };
    case 'LOGIN':
    // ...
  }
  return state;
};
```

Now, let's create a 'Settings' component that will contain our form.
First, let's write the high-level component.

`src/components/Register.js`

```javascript
import ListErrors from './ListErrors';
import React from 'react';
import { Link } from 'react-router';
import agent from '../agent';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  ...state.settings,
  currentUser: state.common.currentUser
});

const mapDispatchToProps = dispatch => ({
  onClickLogout: () => dispatch({ type: 'LOGOUT' }),
  onSubmitForm: user =>
    dispatch({ type: 'SETTINGS_SAVED', payload: agent.Auth.save(user) })
});

class Settings extends React.Component {
  render() {
    return (
      <div className="settings-page">
        <div className="container page">
          <div className="row">
            <div className="col-md-6 offset-md-3 col-xs-12">

              <h1 className="text-xs-center">Your Settings</h1>

              <ListErrors errors={this.props.errors}></ListErrors>

              <SettingsForm
                currentUser={this.props.currentUser}
                onSubmitForm={this.props.onSubmitForm} />

              <hr />

              <button
                className="btn btn-outline-danger"
                onClick={this.props.onClickLogout}>
                Or click here to logout.
              </button>

            </div>
          </div>
        </div>
      </div>
    );
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(Settings);
```

This component can trigger the 2 events that we care about, 'SETTINGS_SAVED'
and 'LOGIN'. However, this 'SettingsForm' component is where the actual
form and internal state will live.

`src/components/Register.js`

```javascript
class SettingsForm extends React.Component {
  constructor() {
    super();

    this.state = {
      image: '',
      username: '',
      bio: '',
      email: '',
      password: ''
    };

    this.updateState = field => ev => {
      const state = this.state;
      const newState = Object.assign({}, state, { [field]: ev.target.value });
      this.setState(newState);
    };

    this.submitForm = ev => {
      ev.preventDefault();

      const user = Object.assign({}, this.state);
      if (!user.password) {
        delete user.password;
      }

      this.props.onSubmitForm(user);
    };
  }

  componentWillMount() {
    if (this.props.currentUser) {
      Object.assign(this.state, {
        image: this.props.currentUser.image || '',
        username: this.props.currentUser.username,
        bio: this.props.currentUser.bio,
        email: this.props.currentUser.email
      });
    }
  }

  componentWillReceiveProps(nextProps) {
    if (nextProps.currentUser) {
      this.setState(Object.assign({}, this.state, {
        image: nextProps.currentUser.image || '',
        username: nextProps.currentUser.username,
        bio: nextProps.currentUser.bio,
        email: nextProps.currentUser.email
      }));
    }
  }

  render() {
    return (
      <form onSubmit={this.submitForm}>
        <fieldset>

          <fieldset className="form-group">
            <input
              className="form-control"
              type="text"
              placeholder="URL of profile picture"
              value={this.state.image}
              onChange={this.updateState('image')} />
          </fieldset>

          <fieldset className="form-group">
            <input
              className="form-control form-control-lg"
              type="text"
              placeholder="Username"
              value={this.state.username}
              onChange={this.updateState('username')} />
          </fieldset>

          <fieldset className="form-group">
            <textarea
              className="form-control form-control-lg"
              rows="8"
              placeholder="Short bio about you"
              value={this.state.bio}
              onChange={this.updateState('bio')}>
            </textarea>
          </fieldset>

          <fieldset className="form-group">
            <input
              className="form-control form-control-lg"
              type="email"
              placeholder="Email"
              value={this.state.email}
              onChange={this.updateState('email')} />
          </fieldset>

          <fieldset className="form-group">
            <input
              className="form-control form-control-lg"
              type="password"
              placeholder="New Password"
              value={this.state.password}
              onChange={this.updateState('password')} />
          </fieldset>

          <button
            className="btn btn-lg btn-primary pull-xs-right"
            type="submit"
            disabled={this.state.inProgress}>
            Update Settings
          </button>

        </fieldset>
      </form>
    );
  }
}
```

See this `updateState` function? React component state doesn't necessarily
have to go through redux. Each component has its own state container, so
we can store the state of the form within the component. All we need to
do is make sure to call the `onSubmitForm` function when the user submits
the form and pass the data out of the component.

There's a few more subtleties here, because the form needs to be
pre-populated with the current user's data. We don't want the form to
be blank when the user loads, so we need 2 lifecycle hooks. First,
on `componentWillMount()`, we'll check if there's a current user and
initialize our state. Second, there's a `componentWillReceiveProps()`
hook, which gets called when the component's 'props' will change. If
the current user changes, we'll update the form with the new information.

### Cleaning Up State

---------------

# Step 7: CRUD Operations For Articles and Comments

### Article View

### Comments List

### CRUD Comments

--------------

# Step 8: Component Inheritance With Profile Page

---------------

# Step 9: Tabs and Pagination

---------------

# Step 10: Editor Form
