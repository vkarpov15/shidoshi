# Step 10: Editor Form

The last view remaining is the actual editor form where you create a new post.
The form needs to also support editting a post as well as editing an
existing post. You also need to be able to add and remove tags for the post
by hitting enter in this input.

### Writing the Editor Component

First, let's add the necessary HTTP methods to `agent.js`. We can already
load an article, but we need to be
able to create an article and update an existing article.

`src/agent.js`

```javascript
const limit = (count, p) => `limit=${count}&offset=${p ? p * count : 0}`;
const encode = encodeURIComponent;
const omitSlug = article => Object.assign({}, article, { slug: undefined });
const Articles = {
  all: page =>
    requests.get(`/articles?${limit(10, page)}`),
  byAuthor: (author, page) =>
    requests.get(`/articles?author=${encode(author)}&${limit(5, page)}`),
  byTag: (tag, page) =>
    requests.get(`/articles?tag=${encode(tag)}&${limit(10, page)}`),
  del: slug =>
    requests.del(`/articles/${slug}`),
  favoritedBy: (author, page) =>
    requests.get(`/articles?favorited=${encode(author)}&${limit(5, page)}`),
  feed: () =>
    requests.get('/articles/feed?limit=10&offset=0'),
  get: slug =>
    requests.get(`/articles/${slug}`),
  update: article =>
    requests.put(`/articles/${article.slug}`, { article: omitSlug(article) }),
  create: article =>
    requests.post('/articles', { article })
};
```

Next, let's write the 'Editor' component.

`src/components/Editor.js`

```javascript
import ListErrors from './ListErrors';
import React from 'react';
import agent from '../agent';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  ...state.editor
});

/**
 * `mapDispatchToProps()` needs separate actions for adding/removing
 * tags, submitting an article, updating individual fields, and cleaning
 * up after navigating away from the page.
 */
const mapDispatchToProps = dispatch => ({
  onAddTag: () =>
    dispatch({ type: 'ADD_TAG' }),
  onLoad: payload =>
    dispatch({ type: 'EDITOR_PAGE_LOADED', payload }),
  onRemoveTag: tag =>
    dispatch({ type: 'REMOVE_TAG', tag }),
  onSubmit: payload =>
    dispatch({ type: 'ARTICLE_SUBMITTED', payload }),
  onUnload: payload =>
    dispatch({ type: 'EDITOR_PAGE_UNLOADED' }),
  onUpdateField: (key, value) =>
    dispatch({ type: 'UPDATE_FIELD_EDITOR', key, value })
});

class Editor extends React.Component {
  constructor() {
    super();

    const updateFieldEvent =
      key => ev => this.props.onUpdateField(key, ev.target.value);
    this.changeTitle = updateFieldEvent('title');
    this.changeDescription = updateFieldEvent('description');
    this.changeBody = updateFieldEvent('body');
    this.changeTagInput = updateFieldEvent('tagInput');

    // When entering tags, hitting enter adds a tag to the list
    this.watchForEnter = ev => {
      if (ev.keyCode === 13) {
        ev.preventDefault();
        this.props.onAddTag();
      }
    };

    this.removeTagHandler = tag => () => {
      this.props.onRemoveTag(tag);
    };

    // When submitting the form, we need to correctly format the
    // object and use the right function - if we have a slug,
    // we're updating an article, otherwise we're creating a new
    // one.
    this.submitForm = ev => {
      ev.preventDefault();
      const article = {
        title: this.props.title,
        description: this.props.description,
        body: this.props.body,
        tagList: this.props.tagList
      };

      const slug = { slug: this.props.articleSlug };
      const promise = this.props.articleSlug ?
        agent.Articles.update(Object.assign(article, slug)) :
        agent.Articles.create(article);

      this.props.onSubmit(promise);
    };
  }

  /**
   * React-router has an interesting quirk: if two routes have the
   * same component, react-router will reuse the component when
   * switching between the two. So if '/editor' and '/editor/slug'
   * both use the 'Editor' component, react-router won't recreate
   * the Editor component if you navigate to '/editor' from '/editor/slug'.
   * To work around this, we need the `componentWillReceiveProps()` hook.
   */
  componentWillReceiveProps(nextProps) {
    if (this.props.params.slug !== nextProps.params.slug) {
      if (nextProps.params.slug) {
        this.props.onUnload();
        return this.props.onLoad(agent.Articles.get(this.props.params.slug));
      }
      this.props.onLoad(null);
    }
  }

  componentWillMount() {
    if (this.props.params.slug) {
      return this.props.onLoad(agent.Articles.get(this.props.params.slug));
    }
    this.props.onLoad(null);
  }

  componentWillUnmount() {
    this.props.onUnload();
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(Editor);
```

### Editor Component Render Function

Next let's write the actual render function

```javascript
class Editor extends React.Component {
render() {
    return (
      <div className="editor-page">
        <div className="container page">
          <div className="row">
            <div className="col-md-10 offset-md-1 col-xs-12">

              <ListErrors errors={this.props.errors}></ListErrors>

              <form>
                <fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control form-control-lg"
                      type="text"
                      placeholder="Article Title"
                      value={this.props.title}
                      onChange={this.changeTitle} />
                  </fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control"
                      type="text"
                      placeholder="What's this article about?"
                      value={this.props.description}
                      onChange={this.changeDescription} />
                  </fieldset>

                  <fieldset className="form-group">
                    <textarea
                      className="form-control"
                      rows="8"
                      placeholder="Write your article (in markdown)"
                      value={this.props.body}
                      onChange={this.changeBody}>
                    </textarea>
                  </fieldset>

                  <fieldset className="form-group">
                    <input
                      className="form-control"
                      type="text"
                      placeholder="Enter tags"
                      value={this.props.tagInput}
                      onChange={this.changeTagInput}
                      onKeyUp={this.watchForEnter} />

                    <div className="tag-list">
                      {
                        (this.props.tagList || []).map(tag => {
                          return (
                            <span className="tag-default tag-pill" key={tag}>
                              <i  className="ion-close-round"
                                  onClick={this.removeTagHandler(tag)}>
                              </i>
                              {tag}
                            </span>
                          );
                        })
                      }
                    </div>
                  </fieldset>

                  <button
                    className="btn btn-lg pull-xs-right btn-primary"
                    type="button"
                    disabled={this.props.inProgress}
                    onClick={this.submitForm}>
                    Publish Article
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
```

Next up, we need to wire up react-router to use this component in `index.js`

`src/index.js`

```javascript
import { Provider } from 'react-redux';
import ReactDOM from 'react-dom';
import React from 'react';
import { Router, Route, IndexRoute, hashHistory } from 'react-router';

import App from './components/App';
import Article from './components/Article';
import Editor from './components/Editor';
import Home from './components/Home';
import Login from './components/Login';
import Profile from './components/Profile';
import ProfileFavorites from './components/ProfileFavorites';
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
        <Route path="@:username/favorites" component={ProfileFavorites} />
        <Route path="editor" component={Editor} />
        <Route path="editor/:slug" component={Editor} />
      </Route>
    </Router>
  </Provider>
), document.getElementById('main'));
```

### Wiring Up Reducers

First up, we need a new editor reducer to handle updating the various
editor form actions.

`src/reducers/editor.js`

```javascript

export default (state = {}, action) => {
  switch (action.type) {
    case 'EDITOR_PAGE_LOADED':
      return {
        ...state,
        articleSlug: action.payload ? action.payload.article.slug : '',
        title: action.payload ? action.payload.article.title : '',
        description: action.payload ? action.payload.article.description : '',
        body: action.payload ? action.payload.article.body : '',
        tagInput: '',
        tagList: action.payload ? action.payload.article.tagList : []
      };
    case 'EDITOR_PAGE_UNLOADED':
      return {};
    case 'ARTICLE_SUBMITTED':
      return {
        ...state,
        inProgress: null,
        errors: action.error ? action.payload.errors : null
      };
    case 'ASYNC_START':
      if (action.subtype === 'ARTICLE_SUBMITTED') {
        return { ...state, inProgress: true };
      }
      break;
    case 'ADD_TAG':
      return {
        ...state,
        tagList: state.tagList.concat([state.tagInput]),
        tagInput: ''
      };
    case 'REMOVE_TAG':
      return {
        ...state,
        tagList: state.tagList.filter(tag => tag !== action.tag)
      };
    case 'UPDATE_FIELD_EDITOR':
      return { ...state, [action.key]: action.value };
  }

  return state;
};
```

Next up, we need to add this reducer to our main reducer:

`src/store.js`

```javascript
import { applyMiddleware, createStore, combineReducers } from 'redux';
import { promiseMiddleware, localStorageMiddleware } from './middleware';
import article from './reducers/article';
import articleList from './reducers/articleList';
import auth from './reducers/auth';
import common from './reducers/common';
import editor from './reducers/editor';
import home from './reducers/home';
import profile from './reducers/profile';
import settings from './reducers/settings';

const reducer = combineReducers({
  article,
  articleList,
  auth,
  common,
  editor,
  home,
  profile,
  settings
});

const middleware = applyMiddleware(promiseMiddleware, localStorageMiddleware);

const store = createStore(reducer, middleware);

export default store;
```

Finally, we also need to wire up the 'common' reducer to redirect the
user to the article view when they successfully submit a new article:

`src/common.js`

```javascript
const defaultState = {
  appName: 'Conduit',
  token: null
};

export default (state = defaultState, action) => {
  switch (action.type) {
    // ...
    case 'LOGOUT':
      return { ...state, redirectTo: '/', token: null, currentUser: null };
    case 'ARTICLE_SUBMITTED':
      const redirectUrl = `article/${action.payload.article.slug}`;
      return { ...state, redirectTo: redirectUrl };
    case 'SETTINGS_SAVED':
    // ...
  }
  return state;
};
```

Now you can submit an article and edit an existing article, and that means
you have a fully functional conduit app. Congratulations!
