# Best Practice: How I structure my React/ Redux projects

When developing frontends I personally love to work with [React](https://reactjs.org/) and [Redux](https://redux.js.org/) due to its functional paradigms: You can easily develop, compose and test web components - Wondeful. But when developing larger projects one have to think about a good structure of components and code artifacts. When it comes to sharing components and (redux-)logic between several project a good composition is keen.

![Best Practice: How I structure my React/ Redux projects](../images/2018-01-26_reactredux.png)

In the following I want to give you insights how I started structuring my projects almost a year ago from now and I still stick to this structure as I made good experience with it throughout the last year.

```topics
web, misc
```

---

A good starting point to develop a React project is Facebook's [React Create App](https://github.com/facebook/create-react-app) it already gives you a good basic project structure and the most important tool integrations. If you don't know it - Have look into the well written docs.

The initial project structure created by React Create App looks as follows:

```
my-app
├── README.md
├── package.json
├── .gitignore
├── public
│   ├── favicon.ico
│   ├── index.html
│   └── manifest.json
└── src
    ├── App.css
    ├── App.js
    ├── App.test.js
    ├── index.css
    ├── index.js
    ├── logo.svg
    └── registerServiceWorker.js
```

When you start introducing Redux you also need places for the typical redux artifacts: the store, reducers, epics and actions. Thus I restructure the directories as follows:

```
my-app
├── README.md
├── package.json
├── .gitignore
├── public
│   ├── favicon.ico
│   ├── index.html
│   └── manifest.json
└── src
    ├── components
    │   └── App
    │     ├── App.css
    │     ├── App.js
    │     ├── App.test.js
    │     └── logo.svg
    ├── elements
    ├── redux
    ├── resources
    ├── stories
    ├── utils
    ├── index.css
    ├── index.js
    ├── registerServiceWorker.js
    └── routes.js
```

The `src` directory now contains a bunch of sub-directories:

  * `components` will include major application components, these components will typically use redux. I'll explain their internal structure in more detail below.

  * `elements` includes basic react components which do not have any relation to redux, but might be used in multiple other elements or components.

  * `redux` will include the redux specific artifacts. Below I will explain the internal structure of this directory.

  * `resources` typically contains language files for [react-intl](https://github.com/yahoo/react-intl)

  * `stories` may include contents for [Storybook](https://github.com/storybooks/storybook)

  * `utils` typically includes some basic functions I'm using in several places of my project.

Let's have a more detailed look into the `redux` directory: The `redux` directory contains actions, reducers and epics for different functionalities. I always try to decouple functionalities as good as possible to be able to exchange components and parts between different applications; and of course to keep maintanance of a larger project as simple as possible. The redux directoy may contain the following structure and artefacts:

```
redux
├── _template
│   ├── actions.js
│   ├── epics.js
│   └── reducers.js
├── app
│   ├── actions.js
│   ├── epics.js
│   └── reducers.js
├── services
│   ├── service-a
│   │   ├── actions.js
│   │   ├── epics.js
│   │   └── reducers.js
│   ├── service-b
│   │   ├── actions.js
│   │   ├── epics.js
│   │   └── reducers.js      
│   ├── actions.js
│   ├── epics.js
│   └── reducers.js
├── actions.js
├── epics.js
├── reducers.js
└── store.js
```

As you can see, each dirctory contains `actions.js`, `epics.js` and `reducers.js`. A leaf directory contains the actual actions, epics and reducers whereas a node-directory within the tree just composes its childs. This allows you to easily introduce new layers in one tree path without changing the layers higher in the directory tree.

Let's have a look into the files to see how the composition works:

  * `action.js` on a leaf defines the actions - It basically provides the action name and helper-method to instantiate an action object to dispatch via redux.

    ```javascript
    import constantsFromArray from '../../utils/constants-from-array';

    export const types = constantsFromArray([
      'FOO',
      'LOREM'
    ], 'BAR_');

    export const foo = (payload) => (
      { type: types.FOO, payload }
    );

    export const lorem = (payload) => (
      { type: types.LOREM, payload }
    );

    export default {
      foo,
      lorem
    }
    ```
    
    **Note:** The function `_.constantsFromArray` simply creates an object where `key === value` or, optionally, `key === prefix + value`. In the example above it creates an object:

    ```javascript
    {
      FOO: 'BAR_FOO',
      LOREM: 'LOREM_FOO'
    }
    ```

    In the end the module exports an object which can be merged into its parent's action object as we'll see later.

  * `epics.js` may contain redux epics, like in the following example:

    ```javascript 
    import 'rxjs';

    import { types } from './actions';

    export const sampleEpic = (action$, store) => action$
      .ofType(types.FOO)
      .mapTo({
        type: types.LOREM,
        payload: {
          foo: 'bar'
        }
      });

    export default [
      sampleEpic
    ]
    ```

    The epic utilizes the types defined in `actions.js` and in the end exports an array of epics for this component.

  * `reducers.js` contains the reducers which may change the store's state based on the actions of this component:

    ```javascript
    import { fromJS } from 'immutable';
    import { types } from './actions';

    export const initialState = fromJS({
      value: "",
      output: "Enter your name and press submit!"
    });

    const foo = state => {
      return state;
    };


    export default (state = initialState, action) => {
      switch (action.type) {
        case types.FOO:
          return foo(state, action.payload);
        default:
          return state;
      }
    };
    ```

    As you can see here, for me it's also best practice to use [Immutable JS](https://facebook.github.io/immutable-js/) for the store. Especially transitions in complex object structures are easier to handle with the API of Immutable JS (another gread framework from Facebook).

This is how the leaf redux directories are designed. As I said, all node-directories then just compose these leaf modules in a way that it doesn't matter how deep the tree might become:

  * A node `action.js` might look as follows:

    ```javascript
    import serviceA, { types as serviceATypes } from './service-a/actions';
    import serviceB, { types as serviceBTypes } from './service-b/actions';

    import _ from 'lodash';

    export const types = {
      serviceA: serviceATypes,
      serviceB: serviceBTypes
    };

    export default {
      serviceA,
      serviceB
    };
    ```

    As you can see, the module again exposes to objects one for the types and another one for the action helper methods. In that manner, it doesn't matter if e.g. `service-a` contains more child-nodes or is already a leaf-node.

  * `epics.js` may compose it's childs straight forward by concetanating an array:

    ```javascript
    import _ from 'lodash';
    import serviceA from './service-a/epics';
    import serviceB from './service-b/epics';

    export default _.concat(
      serviceA,
      serviceB);
    ```

  * Finally reducers are combined by `redux-immutable-combine-reducers`:

    ```javascript
    import combineReducers from 'redux-immutable-combine-reducers';
    import serviceA from './service-a/reducers';
    import serviceB from './service-b/reducers';
    import { fromJS } from 'immutable';


    export default combineReducers(fromJS({
      serviceA,
      serviceB
    }));
    ```

To make the story complete, this is how the store is initialized in `store.js`:

```javascript
import { applyMiddleware, compose, createStore } from 'redux';

import { Iterable } from 'immutable';
import { createEpicMiddleware } from 'redux-observable';
import { routerMiddleware as createRouterMiddleware } from 'react-router-redux';
import { history } from '../routes';
import { init } from './app/actions';
import reduxLoggerFactory from 'redux-logger';
import rootEpic from './epics';
import rootReducer from './reducers';

const epicMiddleware = createEpicMiddleware(rootEpic);

const routerMiddleware = createRouterMiddleware(history);

const reduxLoggerMiddleware = reduxLoggerFactory({ 
  stateTransformer: (state) => {
    if (Iterable.isIterable(state)) {
      return state.toJS();
    };

    return true;
  },
  collapsed: true 
})

const composeEnhancers = (typeof window !== 'undefined' && window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__) || compose;

const store = createStore(rootReducer,
  composeEnhancers(
    applyMiddleware(epicMiddleware, reduxLoggerMiddleware, routerMiddleware)
  )
);

store.dispatch(init());

export default store;
```

What we have seen now is already a nice structue, but let's dig a little bit deeper. As I said in the beginning one important thing for me is to keep components as independent as possible. For this reasons I do not want to reference an other components action types within another. Instead I'm using `app/epics` to link actions.

```javascript
import actions, { types } from '../actions';

import { LOCATION_CHANGE } from 'react-router-redux';
import _ from 'lodash';

export default [
  // LOCATION_CHANGE('/tests') -> services.utterances.READ
  (action$, store$) => action$
    .ofType(LOCATION_CHANGE)
    .filter(action => _.get(action, 'payload.pathname', '') === '/tests')
    .flatMap(action => 
      [ 
        actions.services.serviceA.foo(),
      ])
]
```

Doing things that way each redux-component can use its own domain driven language which makes each component good readable and clean. 

One more thing you may wonder about is how composition works when it comes to UI elements. As of now we have just discussed redux-components which are not directly linked to an UI component - Especially service calls or actions which are triggered by server-side events through sockets may include this kind of actions. But of course you will have actions like `BUTTON_CLICKED`, `CONTENT_UPDATED` or similar which are directly linked to the behaviour of a single component; and you may also want to share this component between projects and package the component in a single module. For this reason my components always also include a set of actions, epics and reducers:

```
components
└── component
    └── Component_A
        ├── redux
        │   ├── actions.js
        │   ├── epics.js
        │   └── reducers.js
        ├── component.js
        ├── index.js
        └── styles.css
```

`component.js` includes a plain react component which is wrapped by `react-redux` within `index.js`:

```javascript
import Component from './component';
import _ from 'lodash';
import { connect } from 'react-redux';

const mapStateToProps = (state) => _.get(state.toJS(), 'foo.bar');

const mapDispatchToProps = (dispatch) => {
  return { };
};

const VisibleComponent = connect(
  mapStateToProps,
  mapDispatchToProps
)(Component);

export default VisibleComponent;
```

These component redux structures may also be with multiple levels, just like the usual ones mentioned above. Within `redux/components` all these component-specific redux-components are bundled to the usual redux-tree.

As mentioned in the beginning this structure was for a really good approach to keep the sources clean and composable. I'm able to reuse components including redux logic over multiple projects without flaws. Let me know by adding a comment if you have feedback or even better approaches.
