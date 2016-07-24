# Step 7: CRUD Operations For Articles and Comments

Next, let's implement the article view. The article view shows you an article
and lets you comment on it. If you're the owner of the article, you can also
edit it or delete it.

### Article View

First off, we need to add the right HTTP calls to the `agent.js` file. For
now, we'll only need 2: getting an article, and getting a list of the
article's comments.

`src/agent.js`

```javascript
// ...

const Articles = {
  all: page =>
    requests.get(`/articles?limit=10`),
  get: slug =>
    requests.get(`/articles/${slug}`)
};

// ...

const Comments = {
  forArticle: slug =>
    requests.get(`/articles/${slug}/comments`)
};

export default {
  Articles,
  Auth,
  Comments,
  setToken: _token => { token = _token; }
};
```

Next, lets write the top-level Article component. There's going to be
a lot of sub-components for the Article view, so let's create a new directory.

`src/Article/index.js`

```javascript
'use strict';

import ArticleMeta from './ArticleMeta';
import CommentContainer from './CommentContainer';
import { Link } from 'react-router';
import React from 'react';
import agent from '../../agent';
import { connect } from 'react-redux';
import marked from 'marked';

const mapStateToProps = state => ({
  ...state.article,
  currentUser: state.common.currentUser
});

const mapDispatchToProps = dispatch => ({
  onLoad: payload =>
    dispatch({ type: 'ARTICLE_PAGE_LOADED', payload }),
  onUnload: () =>
    dispatch({ type: 'ARTICLE_PAGE_UNLOADED' })
});

class Article extends React.Component {
  componentWillMount() {
    this.props.onLoad(Promise.all([
      agent.Articles.get(this.props.params.id),
      agent.Comments.forArticle(this.props.params.id)
    ]));
  }

  componentWillUnmount() {
    this.props.onUnload();
  }

  render() {
    if (!this.props.article) {
      return null;
    }

    const markup = { __html: marked(this.props.article.body) };
    const canModify = this.props.currentUser &&
      this.props.currentUser.username === this.props.article.author.username;
    return (
      <div className="article-page">

        <div className="banner">
          <div className="container">

            <h1>{this.props.article.title}</h1>
            <ArticleMeta
              article={this.props.article}
              canModify={canModify} />

          </div>
        </div>

        <div className="container page">

          <div className="row article-content">
            <div className="col-xs-12">

              <div dangerouslySetInnerHTML={markup}></div>

              <ul className="tag-list">
                {
                  this.props.article.tagList.map(tag => {
                    return (
                      <li
                        className="tag-default tag-pill tag-outline"
                        key={tag}>
                        {tag}
                      </li>
                    );
                  })
                }
              </ul>

            </div>
          </div>

          <hr />

          <div className="article-actions">
          </div>

          <div className="row">
            <CommentContainer
              comments={this.props.comments || []}
              errors={this.props.commentErrors}
              slug={this.props.params.id}
              currentUser={this.props.currentUser} />
          </div>
        </div>
      </div>
    );
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(Article);
```

For this component, `mapStateToProps` will pull all the data from the
`article` reducer that you'll write later, as well as the currently logged
in user.

When loaded, this component will load the comments and article details.

This 'ArticleMeta' component will contain details about the article's author
as well as any actions the user can take on the article, like editting or
deleting.

`marked` is a library that compiles markdown into HTML - in order to get
react to render raw HTML, we need to use this `dangerouslySetInnerHTML`
property, because React sanitizes HTML by default.

Finally there's this `CommentContainer` component that's going to contain
our comments.

### Wiring Up The Reducer

Next, let's add a basic `article` reducer:

`src/reducers/article.js`

```javascript
'use strict';

export default (state = {}, action) => {
  switch (action.type) {
    case 'ARTICLE_PAGE_LOADED':
      return {
        ...state,
        article: action.payload[0].article,
        comments: action.payload[1].comments
      };
      break;
    case 'ARTICLE_PAGE_UNLOADED':
      return {};
  }

  return state;
};
```

And add it to `store.js`:

`src/reducers/store.js`

```javascript
import { applyMiddleware, createStore, combineReducers } from 'redux';
import { promiseMiddleware, localStorageMiddleware } from './middleware';
import article from './reducers/article';
import auth from './reducers/auth';
import common from './reducers/common';
import home from './reducers/home';
import settings from './reducers/settings';

const reducer = combineReducers({
  article,
  auth,
  common,
  home,
  settings
});
// ...
```

Next, let's add this route to react-router setup in `index.js`:

`src/index.js`

```javascript
// ...
import App from './components/App';
import Article from './components/Article';
import Home from './components/Home';
import Login from './components/Login';
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
      </Route>
    </Router>
  </Provider>
), document.getElementById('main'));
```

And modify our `ArticlePreview` component to link to the article view:

```javascript
import React from 'react';

const ArticlePreview = props => {
  const article = props.article;

  return (
    <div className="article-preview">
      <div className="article-meta">
        <Link to={`@${article.author.username}`}>
          <img src={article.author.image} />
        </Link>

        <div className="info">
          <Link className="author" to={`@${article.author.username}`}>
            {article.author.username}
          </Link>
          <span className="date">
            {new Date(article.createdAt).toDateString()}
          </span>
        </div>

        <div className="pull-xs-right">
          <button className="btn btn-sm btn-outline-primary">
            <i className="ion-heart"></i> {article.favoritesCount}
          </button>
        </div>
      </div>

      <Link to={`article/${article.slug}`} className="preview-link">
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
      </Link>
    </div>
  );
}

export default ArticlePreview;
```

Now, if you comment out all references to the not yet written
`CommentContainer` and `ArticleMeta` components, you'll get a simple
view that displays the article's title when you click on an article
on the home page.

### CRUD Articles

That's the basics of displaying the article view. Now let's write the
`ArticleMeta` component. This component will display information about
the author and edit and delete buttons.

`src/Article/ArticleMeta.js`

```javascript
import ArticleActions from './ArticleActions';
import { Link } from 'react-router';
import React from 'react';

const ArticleMeta = props => {
  const article = props.article;
  return (
    <div className="article-meta">
      <Link to={`@${article.author.username}`}>
        <img src={article.author.image} />
      </Link>

      <div className="info">
        <Link to={`@${article.author.username}`} className="author">
          {article.author.username}
        </Link>
        <span className="date">
          {new Date(article.createdAt).toDateString()}
        </span>
      </div>

      <ArticleActions canModify={props.canModify} article={article} />
    </div>
  );
};

export default ArticleMeta;
```

These links are to the profile view, which you'll implement later.
This component also will use the 'ArticleActions' component, which is where
the edit and delete buttons will live. To enable the delete button, we'll
need to add an HTTP call to `agent.js` for deleting articles.

`src/agent.js`

```javascript
// ...

const requests = {
  del: url =>
    superagent.del(`${API_ROOT}${url}`).use(tokenPlugin).then(responseBody),
  get: url =>
    superagent.get(`${API_ROOT}${url}`).use(tokenPlugin).then(responseBody),
  post: (url, body) =>
    superagent.post(`${API_ROOT}${url}`, body).use(tokenPlugin).then(responseBody),
  put: (url, body) =>
    superagent.put(`${API_ROOT}${url}`, body).use(tokenPlugin).then(responseBody)
};

const Articles = {
  all: page =>
    requests.get(`/articles?limit=10`),
  del: slug =>
    requests.del(`/articles/${slug}`),
  get: slug =>
    requests.get(`/articles/${slug}`)
};
// ...
```

Next, let's take a look at the 'ArticleActions' component and implement the
delete button.

`src/Article/ArticleActions.js`

```javascript
import { Link } from 'react-router';
import React from 'react';
import agent from '../../agent';
import { connect } from 'react-redux';

const mapDispatchToProps = dispatch => ({
  onClickDelete: payload =>
    dispatch({ type: 'DELETE_ARTICLE', payload })
});

const ArticleActions = props => {
  const article = props.article;
  const del = () => {
    props.onClickDelete(agent.Articles.del(article.slug))
  };
  if (props.canModify) {
    return (
      <span>

        <Link
          to={`/editor/${article.slug}`}
          className="btn btn-outline-secondary btn-sm">
          <i className="ion-edit"></i> Edit Article
        </Link>

        <button className="btn btn-outline-danger btn-sm" onClick={del}>
          <i className="ion-trash-a"></i> Delete Article
        </button>

      </span>
    );
  }

  return (
    <span>
    </span>
  );
};

export default connect(() => ({}), mapDispatchToProps)(ArticleActions);
```

`mapDispatchToProps()` will expose one action, which will be a 'DELETE_ARTICLE'
action. It'll use the `del` function we just added to agent.

If the user can modify this article, we'll display a link to edit the
article and a button to delete the article. We'll implement this 'Editor'
component later.

Next up, let's handle the 'DELETE_ARTICLE' action in a reducer. The changes
will be in the common reducer, since we want to redirect the user to the
home view after they delete an article.

```javascript
// ...

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
    case 'DELETE_ARTICLE':
      return { ...state, redirectTo: '/' };
  }
  // ...
};
```

Now, you can find an article on the real ProductionReady, open that article
in your local ProductionReady, and delete it.

### CRUD Comments

Next, let's implement the 'CommentContainer' component. 'CommentContainer' is just
going to be a thin wrapper around a 'CommentList' component.

`src/components/Article/CommentContainer.js`

```javascript
import CommentInput from './CommentInput';
import CommentList from './CommentList';
import { Link } from 'react-router';
import React from 'react';

const CommentContainer = props => {
  if (props.currentUser) {
    return (
      <div className="col-xs-12 col-md-8 offset-md-2">
        <div>
          <list-errors errors={props.errors}></list-errors>
          <CommentInput slug={props.slug} currentUser={props.currentUser} />
        </div>

        <CommentList
          comments={props.comments}
          slug={props.slug}
          currentUser={props.currentUser} />
      </div>
    );
  } else {
    return (
      <div className="col-xs-12 col-md-8 offset-md-2">
        <p>
          <Link to="login">Sign in</Link>
          &nbsp;or&nbsp;
          <Link to="register">sign up</Link>
          &nbsp;to add comments on this article.
        </p>

        <CommentList
          comments={props.comments}
          slug={props.slug}
          currentUser={props.currentUser} />
      </div>
    );
  }
};

export default CommentContainer;
```

Mostly what this component will do is show the correct header on top of the
'CommentList' component based on whether the user is logged in. First, let's
implement the 'CommentInput' component. To implement 'CommentInput', first we
need to add an HTTP call to `agent.js` to create a comment:

`src/agent.js`

```javascript
// ...
const Comments = {
  create: (slug, comment) =>
    requests.post(`/articles/${slug}/comments`, { comment }),
  forArticle: slug =>
    requests.get(`/articles/${slug}/comments`)
};
// ...
```

Next, let's put this HTTP call into the `mapDispatchToProps()` function for
the `CommentInput` component.

`src/components/Article/CommentInput.js`

```javascript
import React from 'react';
import agent from '../../agent';
import { connect } from 'react-redux';

const mapDispatchToProps = dispatch => ({
  onSubmit: payload =>
    dispatch({ type: 'ADD_COMMENT', payload })
});

class CommentInput extends React.Component {
  constructor() {
    super();
    this.state = {
      body: ''
    };

    this.setBody = ev => {
      this.setState({ body: ev.target.value });
    };

    this.createComment = ev => {
      ev.preventDefault();
      const payload = agent.Comments.create(this.props.slug,
        { body: this.state.body });
      this.setState({ body: '' });
      this.props.onSubmit(payload);
    };
  }

  render() {
    return (
      <form className="card comment-form" onSubmit={this.createComment}>
        <div className="card-block">
          <textarea className="form-control"
            placeholder="Write a comment..."
            value={this.state.body}
            onChange={this.setBody}
            rows="3">
          </textarea>
        </div>
        <div className="card-footer">
          <img
            src={this.props.currentUser.image}
            className="comment-author-img" />
          <button
            className="btn btn-sm btn-primary"
            type="submit">
            Post Comment
          </button>
        </div>
      </form>
    );
  }
}

export default connect(() => ({}), mapDispatchToProps)(CommentInput);
```

Next let's handle the 'ADD_COMMENT' action in the 'article' reducer:

`src/reducers/article.js`

```javascript
'use strict';

export default (state = {}, action) => {
  switch (action.type) {
    // ...
    case 'ADD_COMMENT':
      return {
        ...state,
        commentErrors: action.error ? action.payload.errors : null,
        comments: action.error ?
          null :
          (state.comments || []).concat([action.payload.comment])
      };
  }

  return state;
};
```

Now that you can create a comment, let's implement the `CommentList` component:

`src/components/Article/CommentList.js`

```javascript
import Comment from './Comment';
import React from 'react';

const CommentList = props => {
  return (
    <div>
      {
        props.comments.map(comment => {
          return (
            <Comment
              comment={comment}
              currentUser={props.currentUser}
              slug={props.slug}
              key={comment.id} />
          );
        })
      }
    </div>
  );
};

export default CommentList;
```

This component really just serves as a wrapper around the 'Comment' component, so let's
implement that.

`src/components/Article/Comment.js`

```javascript
import DeleteButton from './DeleteButton';
import { Link } from 'react-router';
import React from 'react';

const Comment = props => {
  const comment = props.comment;
  const show = props.currentUser &&
    props.currentUser.username === comment.author.username;
  return (
    <div className="card">
      <div className="card-block">
        <p className="card-text">{comment.body}</p>
      </div>
      <div className="card-footer">
        <Link
          to={`@${comment.author.username}`}
          className="comment-author">
          <img src={comment.author.image} className="comment-author-img" />
        </Link>
        &nbsp;
        <Link
          to={`@${comment.author.username}`}
          className="comment-author">
          {comment.author.username}
        </Link>
        <span className="date-posted">
          {new Date(comment.createdAt).toDateString()}
        </span>
        <DeleteButton show={show} slug={props.slug} commentId={comment.id} />
      </div>
    </div>
  );
};

export default Comment;
```

We're going to have links to the author and a delete button that shows up if you're
the one who posted the comment. First let's add an HTTP call to delete a comment to
`agent.js`:

`src/agent.js`

```javascript
// ...
const Comments = {
  create: (slug, comment) =>
    requests.post(`/articles/${slug}/comments`, { comment }),
  delete: (slug, commentId) =>
    requests.del(`/articles/${slug}/comments/${commentId}`),
  forArticle: slug =>
    requests.get(`/articles/${slug}/comments`)
};
// ...
```

Next, let's add the 'DeleteButton' component and call this function in
`mapDispatchToProps()`:

`src/components/Article/DeleteButton.js`

```javascript
import React from 'react';
import agent from '../../agent';
import { connect } from 'react-redux';

const mapDispatchToProps = dispatch => ({
  onClick: (payload, commentId) =>
    dispatch({ type: 'DELETE_COMMENT', payload, commentId })
});

const DeleteButton = props => {
  const del = () => {
    const payload = agent.Comments.delete(props.slug, props.commentId);
    props.onClick(payload, props.commentId);
  };

  if (props.show) {
    return (
      <span className="mod-options">
        <i className="ion-trash-a" onClick={del}></i>
      </span>
    );
  }
  return null;
};

export default connect(() => ({}), mapDispatchToProps)(DeleteButton);
```

Finally, we need to handle the 'DELETE_COMMENT' action in the article reducer:

`src/reducers/article.js`

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    // ...
    case 'DELETE_COMMENT':
      const commentId = action.commentId
      return {
        ...state,
        comments: state.comments.filter(comment => comment.id !== commentId)
      };
  }

  return state;
};
```

Now, when you navigate to an article page, you can write a comment, see it when
the page reloads, and delete it.
