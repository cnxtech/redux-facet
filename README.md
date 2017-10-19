# redux-facet

## Purpose

In Redux, all actions share the same channel. Creating reusable action creators and reducer behaviors is hard, because you need a method with which to associate different results of those actions with different parts of the application. Without a plan to address this, actions and reducers are often duplicated: `createGlobalAlert`, `createUsersAlert`, `createPostsAlert`, etc...

`redux-facet` aims to build a pattern which makes it easy to write one set of actions and one reducer, then reuse that behavior in various 'facets' of your application.

```javascript
import facet, {
  combineFacetReducers
} from 'redux-facet';
import { createStore, combineReducers } from 'redux';
import { Provider } from 'react-redux';
import React from 'react';
import ReactDom from 'react-dom';

import userListReducer from 'reducers/userList';
import alertsReducer from 'reducers/common/alerts';
import alertsActions from 'actions/common/alerts';

/**
 * Creating the root reducer
 */
const reducer = combineFacetReducers({
  users: combineReducers({
    alerts: alertsReducer,
    list: userListReducer,
  }),
  posts: combineReducers({
    alerts: alertsReducer,
  }),
  /* ... */
});

/**
 * Creating the store, as usual
 */
const store = createStore(reducer, {});

/**
 * Creating views for the data
 */
const AlertsView = ({ alerts }) => (
  <div>
    {alerts.map(alert => <div>{alert}</div>)}
  </div>
);
const UsersView = ({ users, alerts, createAlert }) => (
  <div>
    <AlertsView alerts={alerts} />
    <ul>
      {
        users.map(
          user => (
            <li onClick={() => createAlert(`Clicked ${user.name}`)}>{user.name}</li>
          )
        )
      }
    </ul>
  </div>
);
const PostsView = ({ alerts, createAlert }) => (
  <div>
    <AlertsView alerts={alerts} />
    <div>No alerts from users will show up here</div>
  </div>
);

/**
 * Using named facet containers instead of default react-redux containers
 */
const UsersContainer = facet(
  'users',
  (state) => ({ users: state.list, alerts: state.alerts }),
  (dispatch) => ({ createAlert: (message) => dispatch(alertsActions.create(message)) }),
)(UsersView);

const PostsContainer = facet(
  'posts',
  (state) => ({ alerts: state.alerts }),
  (dispatch) => ({ createAlert: (message) => dispatch(alertsActions.create(message)) }),
)(PostsView);

/**
 * Rendering everything
 */
ReactDom.render(
  <Provider store={store}>
    <div>
      <UsersContainer />
      <PostsContainer />
    </div>
  </Provider>
);
```

In the example above, both the `users` and `posts` facets of the application can reuse the same action creators, reducers, and components to manage their alert systems, but the alerts they create will never cross the boundaries between them. `redux-facet` ensures the actions reach the correct reducer, and the state is separated out before reaching the container.

## Documentation

### `facet(facetName: String, baseMapStateToProps: Function, baseMapDispatchToProps: Function, baseMergeProps: Function, options: Object)`

Think of `facet()` kind of like `connect()`. It's a wrapper around `connect` which ensures two things:

1. All actions dispatched by the wrapped component will be tracked with your facet name.
2. The state used to create props will be, by default, only the subsection of the state associated with your facet.

For an action creator,

```javascript
const getUser = (id) => ({
  type: 'GET_USER',
  payload: { id },
});
```

and a given Redux state,

```javascript
{
  users: {
    userId1: { name: 'Bob' },
    userId2: { name: 'Alice' },
  },
  posts: {
    postId1: { content: 'Hello world' },
  },
}
```

using `facet` as follows

```javascript
facet(
  'usersList',
  (state) => { users: state },
  (dispatch) => { getUser: (id) => dispatch(getUser(id)) },
)(Component);
```

will pass the following props to the wrapped component:

```javascript
{
  users: {
    userId1: { name: 'Bob' },
    userId2: { name: 'Alice' },
  },
  getUser: [Function],
}
```

And when component calls `getUser(id)`, the resulting action will look like this:

```javascript
{
  type: 'GET_USER',
  payload: { id },
  meta: { facetName: 'usersList' },
}
```

That's all the unique functionality of `facet()`; the rest is handled by an internal call to `connect` from `react-redux`. If you want to provide options to `connect`, you can pass them in the fourth parameter. Likewise, `mergeProps` is available as the third parameter.

Though simple, `facet()` allows action creators to be written once and reused anywhere without creating ambiguity of which portion of the app generated the action. When coupled with `facetReducer`, this allows actions to be tracked and associated with specific sections of the Redux state.

### facetReducer(facetName: String, reducer: Function)

> Note: for basic usage, be sure to also see `combineFacetReducers` below.

Wrap a reducer in `facetReducer` to restrict it to use only actions which were dispatched from the corresponding facet.

Now that only outgoing actions that are tagged with `facet()` will be processed by this reducer, you are free to extract and reuse the business logic of the reducer funciton itself.

```javascript
// a general reducer that can add and clear alert ids. In this scenario,
// let's suppose that these ids are referencing a normalized collection
// of alerts mounted in our global state.
const alertReducer = (state, action) => {
  switch (action.type) {
    case 'ADD_ALERT':
      return [
        ...state,
        action.payload.alert.id,
      ];
    case 'DISMISS_ALERTS':
      return [];
  }
};

// now the reducer can be reused for various different parts of the app
const rootReducer = combineReducers({
  users: facetReducer('users', combineReducers({
    alerts: alertReducer,
  })),
  posts: facetReducer('posts', combineReducers({
    posts: alertReducer,
  })),
});
```

Unlike if `alertReducer` had been simply mounted as-is, the `posts` will only process `"ADD_ALERT"` events which are related to the `posts` facet. Same with `users`.

#### One Rule: Mount a facet reducer by its name

Presently, `redux-facet` expects a reducer which controls the state of a facet to be mounted at the facet's name. For instance, in the `rootReducer` above, `users` and `posts` are mounted correctly. `combineFacetReducers` was designed to make this more idiomatic.

### combineFacetReducers(reducerMap: Object)

`combineFacetReducers` is the analogue of `combineReducers`. It has the same usage as the default Redux tool. Simply supply it with a map of reducers, where the key for each reducer is the name of the reducer's facet. `combineFacetReducers` will automatically apply `facetReducer` to your reducers utilizing the key and combine them into one function.

```javascript
const rootReducer = combineFacetReducers({
  users: userReducer,
  posts: postReducer,
});
```

### facetSaga(facetName: String, pattern: String|Function, saga: Function)

For those who use `redux-saga`, this library also provides a wrapper for sagas which will filter `take` and `put` effects to only draw from actions tagged with the supplied facet name. The API for `facetSaga` is a bit more advanced than `facetReducer`.

The second parameter, `pattern`, is passed on to `redux-saga`'s `take` effect. It will filter actions *after* the facet filter has already been applied. The presence of this parameter is necessary, even if you just use `'*'` to take all actions, as will be explained further.

The third parameter is a generator function which should contain your saga logic. This function will be passed a parameter, `channel`. `channel` is the source and sink for all facet-related actions.

At this point, an example may be most helpful:

```javascript
const handleListSaga = function*(channel) {
  function* handleEvents(action) {
    const response = yield call(api.fetch, action.payload.details);
    // actions must be put back to the channel to retain facet metadata
    yield put(channel, listActions.listComplete(response));
  }

  // use any 'take' based effect with the channel
  yield takeLatest(channel, handleEvents);
};

facetSaga(
  'users',
  'LIST_USERS_REQUEST',
  handleListSaga,
);
```

`'users'` is the facet name which the saga will be filtered on--only actions with the facet name `'users'` will be taken from the channel.

The `'LIST_USERS_REQUEST'` pattern further narrows the collection of actions this saga will trigger from. This is necessary since `take`-based effects currently do not support using both a `channel` and a `pattern` at the same time.

Finally, a saga is supplied which will receive the `channel` as a parameter. `channel` has been configured to emit actions which match the filter parameters, and calling `put` with `channel` will automatically tag outgoing actions with the facet name.

Note that the saga in this example is generalizeable. Since the outgoing action will be tagged with the facet name, it will also be processed exclusively by the reducer associated with that facet, and therefore the results of the operation will only be stored in the state associated with the facet.

### `getFacet(action: Object)`

Returns the facet name which this action has been tagged with.

### `hasFacet(facetName: String) => (action) => true|false`

A 'thunk' which creates a function which takes an action and returns whether or not the action has the specified facet name. Usage: `hasFacet('users')(someAction)`

### `withFacet(facetName: String) => (action) => taggedAction`

A 'thunk' which creates a function which will tag an action with the specified facet name. General usage will probably be to wrap an action creator. For example: `withFacet('users')(createSomeAction(foo, bar))`
