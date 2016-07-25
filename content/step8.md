# Step 8: Profile View, Component Inheritance

One great feature of React components is that they're plain old JavaScript
classes, which means that your components can inherit from other components.
Let's see this in action in building the profile view. The profile view has
2 tabs - in this case, this 2nd tab will be a separate view that will inherit
from the first one.

### The ArticleList Reducer

For the profile page we're going to need to display a list of the user's
articles. In order to do that, we need to decouple the ArticleList component
from the home view's reducer. Let's write a separate 'articleList' reducer:

`src/reducers/articleList.js`

```javascript
'use strict';

export default (state = {}, action) => {
  switch (action.type) {
    case 'HOME_PAGE_LOADED':
      return {
        ...state,
        articles: action.payload.articles,
        articlesCount: action.payload.articlesCount
      };
    case 'HOME_PAGE_UNLOADED':
      return {};
    case 'PROFILE_PAGE_LOADED':
    case 'PROFILE_FAVORITES_PAGE_LOADED':
      return {
        ...state,
        articles: action.payload[1].articles,
        articlesCount: action.payload[1].articlesCount
      };
    case 'PROFILE_PAGE_UNLOADED':
    case 'PROFILE_FAVORITES_PAGE_UNLOADED':
      return {};
  }

  return state;
};
```

Notice that we're also adding action handlers for the profile view and
the profile favorites view, which is the second view that we'll be writing.
In both cases we're just pulling the list of articles from the payload.

Next, let's change the home reducer so it doesn't do anything with the
'HOME_PAGE_LOADED' action:

`src/reducers/home.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'HOME_PAGE_UNLOADED':
      return {};
  }

  return state;
};
```

Finally, we need to tweak the MainView component to get the articles list from
the 'articleList' reducer rather than the 'home' reducer.

`src/Home/MainView.js`

```javascript
// ...

const mapStateToProps = state => ({
  ...state.articleList
});

// ...
```

Great, now the home view works as expected, and our 'articleList' reducer
is hooked up to work with the Profile view. Let's get started on the
Profile view.

### Profile View

The primary functions of the profile view are to display a user's profile,
their articles, and let you follow/unfollow them. Let's add the corresponding
HTTP calls to `agent.js`:

`src/agent.js`

```javascript
// ...

const Articles = {
  all: page =>
    requests.get(`/articles?limit=10`),
  byAuthor: (author, page) =>
    requests.get(`/articles?author=${encodeURIComponent(author)}&limit=5`),
  del: slug =>
    requests.del(`/articles/${slug}`),
  get: slug =>
    requests.get(`/articles/${slug}`)
};

// ...

const Profile = {
  follow: username =>
    requests.post(`/profiles/${username}/follow`),
  get: username =>
    requests.get(`/profiles/${username}`),
  unfollow: username =>
    requests.del(`/profiles/${username}/follow`)
};

export default {
  Articles,
  Auth,
  Comments,
  Profile,
  setToken: _token => { token = _token; }
};
```

Next, let's write the 'Profile' component:

`src/components/Profile.js`

```javascript
import ArticleList from './ArticleList';
import React from 'react';
import { Link } from 'react-router';
import agent from '../agent';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  ...state.articleList,
  currentUser: state.common.currentUser,
  profile: state.profile
});

const mapDispatchToProps = dispatch => ({
  onFollow: username => dispatch({
    type: 'FOLLOW_USER',
    payload: agent.Profile.follow(username)
  }),
  onLoad: payload => dispatch({ type: 'PROFILE_PAGE_LOADED', payload }),
  onUnfollow: username => dispatch({
    type: 'UNFOLLOW_USER',
    payload: agent.Profile.unfollow(username)
  }),
  onUnload: () => dispatch({ type: 'PROFILE_PAGE_UNLOADED' })
});

class Profile extends React.Component {
  componentWillMount() {
    this.props.onLoad(Promise.all([
      agent.Profile.get(this.props.params.username),
      agent.Articles.byAuthor(this.props.params.username)
    ]));
  }

  componentWillUnmount() {
    this.props.onUnload();
  }

  renderTabs() {
    return (
      <ul className="nav nav-pills outline-active">
        <li className="nav-item">
          <Link
            className="nav-link active"
            to={`@${this.props.profile.username}`}>
            My Articles
          </Link>
        </li>

        <li className="nav-item">
          <Link
            className="nav-link"
            to={`@${this.props.profile.username}/favorites`}>
            Favorited Articles
          </Link>
        </li>
      </ul>
    );
  }

  render() {
    const profile = this.props.profile;
    if (!profile) {
      return null;
    }

    const isUser = this.props.currentUser &&
      this.props.profile.username === this.props.currentUser.username;

    return (
      <div className="profile-page">

        <div className="user-info">
          <div className="container">
            <div className="row">
              <div className="col-xs-12 col-md-10 offset-md-1">

                <img src={profile.image} className="user-img" />
                <h4>{profile.username}</h4>
                <p>{profile.bio}</p>

                <EditProfileSettings isUser={isUser} />
                <FollowUserButton
                  isUser={isUser}
                  user={profile}
                  follow={this.props.onFollow}
                  unfollow={this.props.onUnfollow}
                  />

              </div>
            </div>
          </div>
        </div>

        <div className="container">
          <div className="row">

            <div className="col-xs-12 col-md-10 offset-md-1">

              <div className="articles-toggle">
                {this.renderTabs()}
              </div>

              <ArticleList
                articles={this.props.articles} />
            </div>

          </div>
        </div>

      </div>
    );
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(Profile);
export { Profile as Profile, mapStateToProps as mapStateToProps };
```

Notice this `renderTabs()` function - it's a separate function so the
inheriting class can override it. It also has this link to the profile
favorites view, which we'll write later.

There's two small helper components that we need to add:
the `EditProfileSettings` component, which only shows up when you're looking
at your own profile, and the `FollowUserButton` component, which lets you
follow or unfollow a user.

`src/components/Profile.js`

```javascript
const EditProfileSettings = props => {
  if (props.isUser) {
    return (
      <Link
        to="settings"
        className="btn btn-sm btn-outline-secondary action-btn">
        <i className="ion-gear-a"></i> Edit Profile Settings
      </Link>
    );
  }
  return null;
};

const FollowUserButton = props => {
  if (props.isUser) {
    return null;
  }

  let classes = 'btn btn-sm action-btn';
  if (props.user.following) {
    classes += ' btn-secondary';
  } else {
    classes += ' btn-outline-secondary';
  }

  const handleClick = ev => {
    ev.preventDefault();
    if (props.user.following) {
      props.unfollow(props.user.username)
    } else {
      props.follow(props.user.username)
    }
  };

  return (
    <button
      className={classes}
      onClick={handleClick}>
      <i className="ion-plus-round"></i>
      &nbsp;
      {props.user.following ? 'Unfollow' : 'Follow'} {props.user.username}
    </button>
  );
};
```

### Wiring Up The Reducer

Now, let's add the corresponding profile reducer:

`src/reducers/profile.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'PROFILE_PAGE_LOADED':
      return {
        ...action.payload[0].profile
      };
    case 'PROFILE_PAGE_UNLOADED':
      return {};
    case 'FOLLOW_USER':
    case 'UNFOLLOW_USER':
      return {
        ...action.payload.profile
      };
  }

  return state;
};
```

Let's add this reducer to the main reducer:

`src/store.js`

```javascript
// ...
import common from './reducers/common';
import home from './reducers/home';
import profile from './reducers/profile';
import settings from './reducers/settings';

const reducer = combineReducers({
  article,
  articleList,
  auth,
  common,
  home,
  profile,
  settings
});

// ...
```

And also add the Profile view to `index.js`:

`src/index.js`

```javascript
// ...
import Login from './components/Login';
import Profile from './components/Profile';
import Register from './components/Register';
import Settings from './components/Settings';
import store from './store';

ReactDOM.render((
  <Provider store={store}>
    <Router history={hashHistory}>
      <Route path="/" component={App}>
        <IndexRoute component={Home} />
        <Route path="login" component={Login} />
        <Route path="register" component={Register} />
        <Route path="settings" component={Settings} />
        <Route path="article/:id" component={Article} />
        <Route path="@:username" component={Profile} />
      </Route>
    </Router>
  </Provider>
), document.getElementById('main'));
```

Now, here's the finally functioning profile view.

### Profile Favorites

Once you have the 'Profile' component, the 'ProfileFavorites' component
is easy - we just need to override a couple methods from the 'Profile'
methods.

First off, we need another HTTP method, to get all the articles favorited
by a given user:

`src/agent.js`

```javascript
// ...

const Articles = {
  all: page =>
    requests.get(`/articles?limit=10`),
  byAuthor: (author, page) =>
    requests.get(`/articles?author=${encodeURIComponent(author)}&limit=5`),
  del: slug =>
    requests.del(`/articles/${slug}`),
  favoritedBy: (author, page) =>
    requests.get(`/articles?favorited=${encodeURIComponent(author)}&limit=5`),
  get: slug =>
    requests.get(`/articles/${slug}`)
};
// ...
```

Now, we need to write a component that inherits from the 'Profile' component
that calls this function in the `componentWillMount()` hook.

`src/components/ProfileFavorites.js`

```javascript
import { Profile, mapStateToProps } from './Profile';
import React from 'react';
import { Link } from 'react-router';
import agent from '../agent';
import { connect } from 'react-redux';

const mapDispatchToProps = dispatch => ({
  onFollow: username => dispatch({
    type: 'FOLLOW_USER',
    payload: agent.Profile.follow(username)
  }),
  onLoad: (payload) =>
    dispatch({ type: 'PROFILE_FAVORITES_PAGE_LOADED', payload }),
  onUnfollow: username => dispatch({
    type: 'UNFOLLOW_USER',
    payload: agent.Profile.unfollow(username)
  }),
  onUnload: () =>
    dispatch({ type: 'PROFILE_FAVORITES_PAGE_UNLOADED' })
});

class ProfileFavorites extends Profile {
  componentWillMount() {
    this.props.onLoad(Promise.all([
      agent.Profile.get(this.props.params.username),
      agent.Articles.favoritedBy(this.props.params.username)
    ]));
  }

  componentWillUnmount() {
    this.props.onUnload();
  }

  renderTabs() {
    return (
      <ul className="nav nav-pills outline-active">
        <li className="nav-item">
          <Link
            className="nav-link"
            to={`@${this.props.profile.username}`}>
            My Articles
          </Link>
        </li>

        <li className="nav-item">
          <Link
            className="nav-link active"
            to={`@${this.props.profile.username}/favorites`}>
            Favorited Articles
          </Link>
        </li>
      </ul>
    );
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(ProfileFavorites);
```

We also need to overwrite the `renderTabs()` function, because on this view
the second tab will have the "active" class rather than the first.

We also need to tweak the `profile` reducer to take into account the
'PROFILE_FAVORITES_PAGE_LOADED' action:

`src/reducers/profile.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'PROFILE_PAGE_LOADED':
    case 'PROFILE_FAVORITES_PAGE_LOADED':
      return {
        ...action.payload[0].profile
      };
    case 'PROFILE_PAGE_UNLOADED':
    case 'PROFILE_FAVORITES_PAGE_UNLOADED':
      return {};
    case 'FOLLOW_USER':
    case 'UNFOLLOW_USER':
      return {
        ...action.payload.profile
      };
  }

  return state;
};
```

And now you have a fully functional profile favorites page. You can follow
and unfollow Eric.
