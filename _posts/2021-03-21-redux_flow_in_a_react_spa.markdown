---
layout: post
title:      "Redux Flow in a React SPA"
date:       2021-03-22 00:10:31 +0000
permalink:  redux_flow_in_a_react_spa
---

Redux is a powerful state management package for React. It works by creating a store of data that is then modified through actions that are dispatched to custom reducer functions. Much like how reducer functions return a singular value after taking in an array of data, these custom reducers maintain and return a singular, holistic source of truth (state) for your application.

The user interface listens for some event and dispatches the appropriate action to a reducer. The reducer takes in the action, and returns the updated state to the store after incorporating any necessary changes. The user interface can then leverage the updated store.

There are several advantages to utilizing Redux, but the greatest is a combination between scalability and separation of concerns. Assuming state is properly hoisted, components in a traditional web application using React state management can quickly become tightly coupled to each other, becoming heavily reliant on props passed through several layers nested within each other. Redux eliminates this issue by providing centralized state management through a "store," as well as allowing individual components to easily access this store via a wrapper function. Redux is also conducive to vertical architecture, allowing you to maintain scalability with not much additional set up. 

Preparing Redux for vertical architecture can be done in a few simple steps.

First, we create the store in a dedicated `store.js` file, as opposed to directly within the `index.js` file. Redux allows you to easily incorporate additional middleware, and it can clutter the `index.js` file quickly, which already has a good bit of code in it. By separating out our store into a separate file, we can write code that is easier to read.

```
// src/store.js

import { createStore, applyMiddleware, compose } from 'redux'
import thunk from 'redux-thunk'
import rootReducer from './reducers/root'

const enhancers = [
  applyMiddleware(thunk),
  window.__REDUX_DEVTOOLS_EXTENSION__ && window.__REDUX_DEVTOOLS_EXTENSION__(), 
]

const store = createStore(rootReducer, compose(...enhancers))

export default store
```

In this file, I am also going ahead and importing `redux-thunk` middleware to assist with asynchronous actions and a `rootReducer` which we haven't created yet, but will discuss later. Further, we are incorporating the Redux Dev Tools extension, an invaluable asset in debugging your Redux workflow.

**Note: You must install the Redux Dev Tools extension in your browser before trying to run this code. It will throw an error otherwise.**

`compose` allows us to combine multiple enhancers at once. If we need to add additional enhancers in the future, we can easily do so by appending them to the `enhancers` array.

We can now import this `store` into our `index.js` file.

```
// src/index.js
// . . .
import { Provider } from 'react-redux'
import store from './store'

ReactDOM.render(
  <Provider store={store}>
  	<React.StrictMode>
  		<App />
  	</React.StrictMode>
  </Provider>,
	document.getElementById('root')
)
```

Our React application now has access to our newly created `store`. From here, we can create our `rootReducer`, which will be composed of whichever reducer functions we deem necessary for the functionality of our application. Before creating this file, however, it is a good idea to do a bit a brief planning on the overall structure of the application. Do not get too into the details, but it is helpful to have a general idea of the separation of concerns to know where to begin creating your `rootReducer`.

We can now construct our `rootReducer`.

```
// src/reducers/root.js
import { combineReducers } from 'redux'
import profileReducer from './profile.js'
import postsReducer from './posts.js'

const rootReducer = combineReducers({
  profile: profileReducer,
  posts: postsReducer,
})

export default rootReducer
```

As we develop our application and discover the need for additional reducers, we can simply reference them in this holistic `rootReducer` function. Later, when we need to access state managed by these reducers, we'll be able to access them such as: `state.profile` and `state.posts`. For now, let's build the basic structure of our reducers. These individual reducer functions will all follow a very similar format. Let's start with the bones of our `profileReducer`. Keep good practices in scalability by establishing a strong directory structure early on. This will make it easy to find things later.

Every time our reducers are called by Redux, they will take in two arguments: state, and an action. To initialize state upon our application load, we will provide a default argument for the state parameter. Inside of the function, we can utilize a switch statement to handle different actions that may be dispatched to our reducer. The actions, which we'll shortly discuss, will be a plain, ol' JavaScript object (POJO), with at least one attribute: type. More likely than not, it will also contain some type of payload as well. We will use the `action.type` to engage our switch statement.

```
const profileReducer = (state = { data: {}, isLoading: false, errors: [] }, action) => {
  switch(action.type) {
    case "NEW_PROFILE_REQUEST":
      return { ...state, isLoading: true }
    case "PROFILE_REQUEST_FAILURE":
      return { ...state, errors: action.payload, isLoading: false }
    case "READ_PROFILE_SUCCESS":
      return { ...state, data: action.payload, isLoading: false }
    default:
      return state
  }
}
```

The switch statement checks the type attribute on our dispatched action, and responds with the appropriate state object, potentially incorporating the payload of our action as well. This state is returned to the store, which can then trigger a change in the UI for our actor, for instance.

How do we dispatch actions to our reducers, however? There are several ways we can do this, and I like to utilize modern React workflows using hooks to accomplish these goals. Redux comes with several React hooks out of the box, and they make it easy to tie any individual component to our centralized store, regardless of how deeply they are nested.

Let's say we have a `Navbar` component, which is responsible for maintaining our profile data since it is present on all pages in our application. We can easily connect this component to our store using the Redux `useDispatch` and `useSelector` hooks.

```
// src/components/Navbar.js
import { useSelector, useDispatch } from 'react-redux'
import { useEffect } from 'react'
import readProfile from '../actions/readProfile'

const Navbar = () => {
  const profile = useSelector(state => state.profile.data)
  const dispatch = useDispatch()
  
  useEffect(() => {
    dispatch(readProfile())
  }, [dispatch])
  
  return (
    // . . .
  )
}

export default Navbar
```

`useSelector` lets us access the data in our centralized Redux store. `useDispatch` lets us dispatch actions to our `rootReducer`. You'll note that we aren't explicitly dispatching a POJO to our `rootReducer`, however, despite saying that actions should be an object, with at least a type attribute. This is because we are leveraging our `redux-thunk` middleware in this instance to handle an asynchronous request. Below is the `readProfile` function, and you should be able to see how this works.

```
// src/actions/readProfile.js

const readProfile = () => {
  return async dispatch => {
    dispatch({ type: "NEW_PROFILE_REQUEST" })
    try {
      const response = await fetch(`${process.env.API_ENDPOINT}`)
      const json = await response.json()
      if (json.status === 200) {
        dispatch({ type: "READ_PROFILE_SUCCCESS", payload: json.data })
      } else {
        throw new Error(json.errors)
      }
    } catch (errors) {
      dispatch({ type: "PROFILE_REQUEST_FAILURE", payload: errors })
    }
  }
}

export default readProfile
```

`redux-thunk` allows us to return an asynchronous dispatch function that can make multiple regular dispatches to update our Redux store throughout the process of action itself. For the above flow, we dispatch an initial action of type `"NEW_PROFILE_REQUEST"`, which you may remember will return a state, with the `isLoading` attribute set to `true`. We can use this to conditionally display some kind of loading component in our UI. From there, we make our API call, and conditionally dispatch a success or failure action. These actions correspond to the cases in our `profileReducer`, allowing us finite control over our Redux store from any component in our DOM tree.

As your web application grows in scope, you begin to appreciate the power and convenience of Redux, especially when paired with enhancers such as `redux-thunk`. To be able to quickly and easily access a centralized data store from anywhere in our application is a boon for separation of concerns and scalability.
