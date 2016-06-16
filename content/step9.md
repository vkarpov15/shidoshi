# Step 9: Tabs and Pagination

Next up is filling out the functionality on the home page. When you're logged
in, you don't see the global feed, you instead see a feed from users that you
follow. Furthermore, you can filter posts by tag. And when there are too many
posts in the list, you get multiple pages.

### Tabs

First off, let's implement these tabs on the home page. To get the user feed,
you need to add a new method to `agent.js`:

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
  feed: () =>
    requests.get('/articles/feed?limit=10'),
  get: slug =>
    requests.get(`/articles/${slug}`)
};
// ...
```

Next, let's add 2 helper components to the 'MainView' component, one for each
tab.

`src/components/home/MainView.js`

```javascript
import ArticleList from '../ArticleList';
import React from 'react';
import agent from '../../agent';
import { connect } from 'react-redux';

const YourFeedTab = props => {
  if (props.token) {
    const clickHandler = ev => {
      ev.preventDefault();
      props.onTabClick('feed', agent.Articles.feed());
    }

    return (
      <li className="nav-item">
        <a  href=""
            className={ props.tab === 'feed' ? 'nav-link active' : 'nav-link' }
            onClick={clickHandler}>
          Your Feed
        </a>
      </li>
    );
  }
  return null;
};

const GlobalFeedTab = props => {
  const clickHandler = ev => {
    ev.preventDefault();
    props.onTabClick('all', agent.Articles.all());
  };
  return (
    <li className="nav-item">
      <a
        href=""
        className={ props.tab === 'all' ? 'nav-link active' : 'nav-link' }
        onClick={clickHandler}>
        Global Feed
      </a>
    </li>
  );
};

const mapStateToProps = state => ({
  ...state.articleList,
  token: state.common.token
});

const mapDispatchToProps = dispatch => ({
  onTabClick: (tab, payload) => dispatch({ type: 'CHANGE_TAB', tab, payload })
});

const MainView = props => {
  return (
    <div className="col-md-9">
      <div className="feed-toggle">
        <ul className="nav nav-pills outline-active">

          <YourFeedTab
            token={props.token}
            tab={props.tab}
            onTabClick={props.onTabClick} />

          <GlobalFeedTab tab={props.tab} onTabClick={props.onTabClick} />

        </ul>
      </div>

      <ArticleList
        articles={props.articles} />
    </div>
  );
};

export default connect(mapStateToProps, mapDispatchToProps)(MainView);
```

The YourFeedTab component will only show up if there's a token defined,
that is, if the user is actually logged in. It'll also take in the current
tab from the state, and use that to determine whether it should have the
'active' CSS class.

Both tabs are going to dispatch an 'CHANGE_TAB' action, which will change
the current tab and attach a promise for the promise middleware to resolve.

In `mapDispatchToProps()`, we'll add a `onTabClick` function that will
dispatch the 'CHANGE_TAB' action.

Now, let's add a handler for the 'CHANGE_TAB' action in the 'articleList'
reducer.

`src/reducers/articleList.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    // ...
    case 'CHANGE_TAB':
      return {
        ...state,
        articles: action.payload.articles,
        articlesCount: action.payload.articlesCount,
        tab: action.tab
      };
    case 'PROFILE_PAGE_LOADED':
    case 'PROFILE_FAVORITES_PAGE_LOADED':
    // ...
  }

  return state;
};
```

We also need to modify the `Home` component to initialize the current tab:

`src/components/Home/index.js`

```javascript
// ...

const mapStateToProps = state => ({
  appName: state.common.appName,
  token: state.common.token
});

const mapDispatchToProps = dispatch => ({
  onLoad: (tab, payload) =>
    dispatch({ type: 'HOME_PAGE_LOADED', tab, payload }),
  onUnload: () =>
    dispatch({  type: 'HOME_PAGE_UNLOADED' })
});

class Home extends React.Component {
  componentWillMount() {
    const tab = this.props.token ? 'feed' : 'all';
    const articlesPromise = this.props.token ?
      agent.Articles.feed() :
      agent.Articles.all();

    this.props.onLoad(tab, articlesPromise);
  }

  // ...
}
// ...
```

So now you can go to the home page and change tabs. However, why does it show
the global feed tab initially? The answer is that there's a race condition:
`props.token` isn't set until the 'APP_LOAD' action gets handled, so we need
to modify the 'App' component to not render the current view until it has
managed to fetch the currently logged in user.

`src/components/App.js`

```javascript
// ...

const mapStateToProps = state => ({
  appLoaded: state.common.appLoaded,
  appName: state.common.appName,
  currentUser: state.common.currentUser,
  redirectTo: state.common.redirectTo
});

// ...

class App extends React.Component {
  // ...
  render() {
    if (this.props.appLoaded) {
      return (
        <div>
          <Header
            appName={this.props.appName}
            currentUser={this.props.currentUser} />
          {this.props.children}
        </div>
      );
    }
    return (
      <div>
        <Header
          appName={this.props.appName}
          currentUser={this.props.currentUser} />
      </div>
    );
  }
}
// ...
```

Now the tabs work as expected, you can switch back and forth.

### Tags

There's some tab-like functionality when you click on the tags on the home
page. If you click on a tab, notice that this special new tab pops up.
Let's make it so 'MainView' loads a list of tags and supports filtering by
tag.

First, we need to add 2 HTTP methods to `agent.js`: one to get all the tags,
and one to load articles by tag.

`src/agent.js`

```javascript
// ...

const Articles = {
  all: page =>
    requests.get(`/articles?limit=10`),
  byAuthor: (author, page) =>
    requests.get(`/articles?author=${encodeURIComponent(author)}&limit=5`),
  byTag: (tag, page) =>
    requests.get(`/articles?tag=${encodeURIComponent(tag)}&limit=10`),
  del: slug =>
    requests.del(`/articles/${slug}`),
  favoritedBy: (author, page) =>
    requests.get(`/articles?favorited=${encodeURIComponent(author)}&limit=5`),
  feed: () =>
    requests.get('/articles/feed?limit=10'),
  get: slug =>
    requests.get(`/articles/${slug}`)
};

// ...

const Tags = {
  getAll: () => requests.get('/tags')
};

export default {
  Articles,
  Auth,
  Comments,
  Profile,
  Tags,
  setToken: _token => { token = _token; }
};
```

Next, let's tweak the 'Home' component so it loads the list of tags when
it mounts.

`src/components/Home/index.js`

```javascript
// ...
import Tags from './Tags';
import agent from '../../agent';
import { connect } from 'react-redux';

const Promise = global.Promise;

const mapStateToProps = state => ({
  ...state.home,
  appName: state.common.appName,
  token: state.common.token
});

const mapDispatchToProps = dispatch => ({
  onClickTag: (tag, payload) =>
    dispatch({ type: 'APPLY_TAG_FILTER', tag, payload }),
  onLoad: (tab, payload) =>
    dispatch({ type: 'HOME_PAGE_LOADED', tab, payload }),
  onUnload: () =>
    dispatch({  type: 'HOME_PAGE_UNLOADED' })
});

class Home extends React.Component {
  componentWillMount() {
    const tab = this.props.token ? 'feed' : 'all';
    const articlesPromise = this.props.token ?
      agent.Articles.feed() :
      agent.Articles.all();

    this.props.onLoad(tab, Promise.all([agent.Tags.getAll(), articlesPromise]));
  }

  // ...

  render() {
    return (
      <div className="home-page">

        <Banner token={this.props.token} appName={this.props.appName} />

        <div className="container page">
          <div className="row">
            <MainView />

            <div className="col-md-3">
              <div className="sidebar">

                <p>Popular Tags</p>

                <Tags
                  tags={this.props.tags}
                  onClickTag={this.props.onClickTag} />

              </div>
            </div>
          </div>
        </div>

      </div>
    );
  }
}
// ...
```

We're going to dispatch this action 'APPLY_TAG_FILTER' whenever somebody
clicks on a tab, so we can update the list of articles.

We'll use a separate component called 'Tags' to render the list of tags.

`src/components/Home/Tags.js`

```javascript
import React from 'react';
import agent from '../../agent';

const Tags = props => {
  const tags = props.tags;
  if (tags) {
    return (
      <div className="tag-list">
        {
          tags.map(tag => {
            const handleClick = ev => {
              ev.preventDefault();
              props.onClickTag(tag, agent.Articles.byTag(tag));
            };

            return (
              <a
                href=""
                className="tag-default tag-pill"
                key={tag}
                onClick={handleClick}>
                {tag}
              </a>
            );
          })
        }
      </div>
    );
  } else {
    return (
      <div>Loading Tags...</div>
    );
  }
};

export default Tags;
```

Next, let's handle the modified 'HOME_PAGE_LOADED' action and new
'APPLY_TAG_FILTER' action in the articleList reducer.

`src/reducers/articleList.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'HOME_PAGE_LOADED':
      return {
        ...state,
        articles: action.payload[1].articles,
        articlesCount: action.payload[1].articlesCount,
        tab: action.tab
      };
    // ...
    case 'APPLY_TAG_FILTER':
      return {
        ...state,
        articles: action.payload.articles,
        articlesCount: action.payload.articlesCount,
        tab: null,
        tag: action.tag
      };
    case 'PROFILE_PAGE_LOADED':
    case 'PROFILE_FAVORITES_PAGE_LOADED':
    // ...
  }

  return state;
};
```

And modify the home reducer to get the list of tags:

`src/reducers/home.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'HOME_PAGE_LOADED':
      return {
        ...state,
        tags: action.payload[0].tags
      };
    case 'HOME_PAGE_UNLOADED':
      return {};
  }

  return state;
};
```

Finally, let's hook up the 'MainView' component to display this tag.
We're going to add a new tab component called 'TagFilterTab'

`src/components/Home/MainView.js`

```javascript
// ...

const TagFilterTab = props => {
  if (!props.tag) {
    return null;
  }

  return (
    <li className="nav-item">
      <a href="" className="nav-link active">
        <i className="ion-pound"></i> {props.tag}
      </a>
    </li>
  );
};

// ...

const MainView = props => {
  return (
    <div className="col-md-9">
      <div className="feed-toggle">
        <ul className="nav nav-pills outline-active">

          <YourFeedTab
            token={props.token}
            tab={props.tab}
            onTabClick={props.onTabClick} />

          <GlobalFeedTab tab={props.tab} onTabClick={props.onTabClick} />

          <TagFilterTab tag={props.tag} />

        </ul>
      </div>

      <ArticleList
        articles={props.articles} />
    </div>
  );
};

export default connect(mapStateToProps, mapDispatchToProps)(MainView);
```

### Pagination

Finally, we need to be able to switch between pages when there's a lot
of articles in an article list. First, we need to tweak our HTTP calls to
be able to specify a page:

`src/agent.js`

```javascript
// ...
const limit = (count, p) => `limit=${count}&offset=${p ? p * count : 0}`;
const encode = encodeURIComponent;
const Articles = {
  all: page =>
    requests.get(`/articles?${limit(10, page)}`),
  byAuthor: (author, page) =>
    requests.get(`/articles?author=${encode(author)}&${limit(10, page)}`),
  byTag: (tag, page) =>
    requests.get(`/articles?tag=${encode(tag)}&${limit(10, page)}`),
  del: slug =>
    requests.del(`/articles/${slug}`),
  favoritedBy: (author, page) =>
    requests.get(`/articles?favorited=${encode(author)}&${limit(10, page)}`),
  feed: page =>
    requests.get(`/articles/feed?${limit(10, page)}`),
  get: slug =>
    requests.get(`/articles/${slug}`)
};
// ...
```

Next, let's add a pagination component to the 'ArticleList' component. This
component will be responsible for dispatching a 'SET_PAGE' action that makes
the correct HTTP call.

`src/components/ListPagination.js`

```javascript
import React from 'react';

const ListPagination = props => {
  if (props.articlesCount <= 10) {
    return null;
  }

  const range = [];
  for (let i = 0; i < Math.ceil(props.articlesCount / 10); ++i) {
    range.push(i);
  }

  const setPage = page => props.onSetPage(page);

  return (
    <nav>
      <ul className="pagination">

        {
          range.map(v => {
            const isCurrent = v === props.currentPage;
            const onClick = ev => {
              ev.preventDefault();
              setPage(v);
            };
            return (
              <li
                className={ isCurrent ? 'page-item active' : 'page-item' }
                onClick={onClick}
                key={v.toString()}>

                <a className="page-link" href="">{v + 1}</a>

              </li>
            );
          })
        }

      </ul>
    </nav>
  );
};

export default ListPagination;
```

This `onSetPage()` function will be passed in as a prop to this component,
because each view will need to make a separate HTTP call. Let's put the
'ListPagination' component into the 'ArticleList' component:

`src/components/ArticleList.js`

```javascript
import ArticlePreview from './ArticlePreview';
import ListPagination from './ListPagination';
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
            <ArticlePreview article={article} />
          );
        })
      }

      <ListPagination
        articlesCount={props.articlesCount}
        currentPage={props.currentPage}
        onSetPage={props.onSetPage} />
    </div>
  );
};

export default ArticleList;
```

So the 'ArticleList' component now needs an 'onSetPage' prop. Let's add this
to the 3 components we have that use 'ArticleList': 'MainView', 'Profile', and
'ProfileFavorites'.

`src/javascript/MainView.js`

```javascript
// ...

const mapDispatchToProps = dispatch => ({
  onSetPage: (tab, p) => dispatch({
    type: 'SET_PAGE',
    page: p,
    payload: tab === 'feed' ? agent.Articles.feed(p) : agent.Articles.all(p)
  }),
  onTabClick: (tab, payload) => dispatch({ type: 'CHANGE_TAB', tab, payload })
});

const MainView = props => {
  const onSetPage = page => props.onSetPage(props.tab, page);
  return (
    <div className="col-md-9">
      <div className="feed-toggle">
        <ul className="nav nav-pills outline-active">

          <YourFeedTab
            token={props.token}
            tab={props.tab}
            onTabClick={props.onTabClick} />

          <GlobalFeedTab tab={props.tab} onTabClick={props.onTabClick} />

          <TagFilterTab tag={props.tag} />

        </ul>
      </div>

      <ArticleList
        articles={props.articles}
        articlesCount={props.articlesCount}
        currentPage={props.currentPage}
        onSetPage={onSetPage} />
    </div>
  );
};

export default connect(mapStateToProps, mapDispatchToProps)(MainView);
```

`src/javascript/Profile.js`

```javascript
// ...

const mapDispatchToProps = dispatch => ({
  onFollow: username => dispatch({
    type: 'FOLLOW_USER',
    payload: agent.Profile.follow(username)
  }),
  onLoad: payload => dispatch({ type: 'PROFILE_PAGE_LOADED', payload }),
  onSetPage: (page, payload) => dispatch({ type: 'SET_PAGE', page, payload }),
  onUnfollow: username => dispatch({
    type: 'UNFOLLOW_USER',
    payload: agent.Profile.unfollow(username)
  }),
  onUnload: () => dispatch({ type: 'PROFILE_PAGE_UNLOADED' })
});

class Profile extends React.Component {
  // ...

  onSetPage(page) {
    const promise = agent.Articles.byAuthor(this.props.profile.username, page);
    this.props.onSetPage(page, promise);
  }

  render() {
    const profile = this.props.profile;
    if (!profile) {e
      return null;
    }

    const isUser = this.props.currentUser &&
      this.props.profile.username === this.props.currentUser.username;

    const onSetPage = page => this.onSetPage(page)

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
                articles={this.props.articles}
                articlesCount={this.props.articlesCount}
                currentPage={this.props.currentPage}
                onSetPage={onSetPage} />
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

Notice that `onSetPage` is a separate function, so the 'ProfileFavorites'
component can override it.

`src/components/ProfileFavorites`

```javascript
// ...

const mapDispatchToProps = dispatch => ({
  onFollow: username => dispatch({
    type: 'FOLLOW_USER',
    payload: agent.Profile.follow(username)
  }),
  onLoad: (payload) =>
    dispatch({ type: 'PROFILE_FAVORITES_PAGE_LOADED', payload }),
  onSetPage: (page, payload) => dispatch({ type: 'SET_PAGE', page, payload }),
  onUnfollow: username => dispatch({
    type: 'UNFOLLOW_USER',
    payload: agent.Profile.unfollow(username)
  }),
  onUnload: () =>
    dispatch({ type: 'PROFILE_FAVORITES_PAGE_UNLOADED' })
});

class ProfileFavorites extends Profile {
  // ...

  onSetPage(page) {
    const promise =
      agent.Articles.favoritedBy(this.props.profile.username, page);
    this.props.onSetPage(page, promise);
  }

  // ...
}

export default connect(mapStateToProps, mapDispatchToProps)(ProfileFavorites);
```

Finally, we need to tweak the 'articleList' reducer to set the 'currentPage'
property properly and handle the 'SET_PAGE' action.

`src/reducers/articleList.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case 'HOME_PAGE_LOADED':
      return {
        ...state,
        articles: action.payload[1].articles,
        articlesCount: action.payload[1].articlesCount,
        tab: action.tab,
        currentPage: 0
      };
    case 'HOME_PAGE_UNLOADED':
      return {};
    case 'CHANGE_TAB':
      return {
        ...state,
        articles: action.payload.articles,
        articlesCount: action.payload.articlesCount,
        tab: action.tab,
        tag: null,
        currentPage: 0
      };
    case 'SET_PAGE':
      return {
        ...state,
        articles: action.payload.articles,
        articlesCount: action.payload.articlesCount,
        currentPage: action.page
      };
    case 'APPLY_TAG_FILTER':
      return {
        ...state,
        articles: action.payload.articles,
        articlesCount: action.payload.articlesCount,
        tab: null,
        tag: action.tag,
        currentPage: 0
      };
    case 'PROFILE_PAGE_LOADED':
    case 'PROFILE_FAVORITES_PAGE_LOADED':
      return {
        ...state,
        articles: action.payload[1].articles,
        articlesCount: action.payload[1].articlesCount,
        currentPage: 0
      };
    case 'PROFILE_PAGE_UNLOADED':
    case 'PROFILE_FAVORITES_PAGE_UNLOADED':
      return {};
  }

  return state;
};
```

Now, you can navigate between pages on both the global feed and the
profile favorites view.
