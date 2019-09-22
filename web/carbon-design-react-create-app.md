# Getting Started with Carbon Design and React Create App

In this short post I'd like to describe the steps which are required to use the [Carbon Design System](https://www.carbondesignsystem.com/) within your React app, after you created it with [React Create App](https://github.com/facebook/create-react-app).

```topics
web
```

---

First we initialize our React App as usual:

```bash
$ npx create-react-app my-app
$ cd my-app
```

No we install the required dependencies for Carbon Design:

```bash
$ npm install -S carbon-components-react carbon-components carbon-icons
```

Since Carbon Design styles are provied as `*.scss`-files we also need to install Sass in our project:

```bash
$ npm install -S node-sass
```

Now we are ready to use Carbon Design Components in our application, replace the contents of `src/App.js` with:

```javascript
import 'carbon-components/scss/globals/scss/styles.scss';

import React from 'react';
import logo from './logo.svg';
import './App.css';

import { Content, Header, HeaderName } from 'carbon-components-react';

function App() {
  return (
    <div className="App">
      <Header aria-label="IBM Platform Name">
        <HeaderName href="#" prefix="Your">
          App
        </HeaderName>
      </Header>

      <Content>
        <img src={logo} className="App-logo" alt="logo" />
        <p>
          Edit <code>src/App.js</code> and save to reload. Helloooo!
        </p>
        <a
            className="App-link"
            href="https://reactjs.org"
            target="_blank"
            rel="noopener noreferrer"
        >
          Learn React</a> - <a
            className="App-link"
            href="https://www.carbondesignsystem.com/"
            target="_blank"
            rel="noopener noreferrer"
        >
          Learn Carbon Design System
        </a>
      </Content>
    </div>
  );
}

export default App;
```

**Note:** The import of the `scss`-files is required only once in ypur project, thus if you create other react components in other files, you don't need to re-import `carbon-components/scss/globals/scss/styles.scss`. The root of your components, e.g. `App.js` is a good place for this import.

Finally we can start our initial app:

```bash
$ npm start
```

Cool. If you want to see which components are available in Carbon Design, checkout the available [Carbon Design Storybook](http://react.carbondesignsystem.com/).