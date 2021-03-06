# Server

We start with the main script of our server side app, `server.js`:

```javascript
'use strict';
require('mithril/test-utils/browserMock')(global);
global.window.XMLHttpRequest = require('w3c-xmlhttprequest').XMLHttpRequest;

const server = require('./app/server');
const port = process.env.PORT || 3000;


server.listen(port, () => {
    console.log('Listening on localhost:' + port + '...');
});
```

We need this line:

```javascript
require('mithril/test-utils/browserMock')(global);
```

to make Mithril work on the server side, mocking the browser presence.

We continue building the entry point of the server side app (`app/server/index.js`), initializing the *Express* app:

```javascript
const express = require('express');
const m = require('mithril');
const routes = require('../routes.js');
const toHTML = require('mithril-node-render');

const app = express();
const router = express.Router();
```

After that we are going to add an entry in the *Express*-router for each entry of the routes we previously defined 
in `app/routes.js`.

We have previously mapped a [*Mithril* component](http://mithril.js.org/hyperscript.html#components) to each route. 
We might need to fetch data in the component controller (its `oninit` method) before rendering the page server side, and 
of course we don't want to send the response until the data is fetched. To make this work we build our *Mithril* main 
components as async, returning a promise (more details on this later). On each route match *Express* will resolve the 
promise, and the result of this is passed forward as the response.

```javascript
// Build Express routes mapping Mithril routes
Object.keys(routes).forEach((route) => {

    router.get(route, (req, res, next) => {
        const module = routes[route];
        const onmatch = module.onmatch || (() => module);
        const render = module.render || (a => a);

        const attrs = Object.assign({}, req.params, req.query);

        Promise.resolve()
            .then(() => m(onmatch(attrs, req.url) || 'div', attrs))
            .then(render)
            .then(toHTML)
            .then((html) => {
                res.type('html').send(html);
            })
            .catch(next);
    });
});
```

Let's take a more detailed look at this part of the code. First we get the right main component matching the selected route:

```javascript
const module = routes[route];
```

The following lines take in count the possible presence of a 
[RouteResolver object](http://mithril.js.org/route.html#advanced-component-resolution) as main component, and 
consequently its `onmatch()` and/or `render()` methods - otherwise, it will just return the main module itself:

```javascript
const onmatch = module.onmatch || (() => module);
const render = module.render || (a => a);
```

We are ready to resolve the promise of the the async component, passing down the attrs that come from the *Express* `req` 
object, and render the `view` of the component:

```javascript
const attrs = Object.assign({}, req.params, req.query);

Promise.resolve()
    .then(() => m(onmatch(attrs, req.url) || 'div', attrs))
    .then(render)
```

At this point of the Promise chain we have the [virtual DOM](http://mithril.js.org/vnodes.html) generated by the 
component `view`. Here it comes [mithril-node-render](https://github.com/StephanHoyer/mithril-node-render), which is the 
key to transform that into HTML markup, as a string to pass down to *Express* `res.send` function.


```javascript
    .then(toHTML)
    .then((html) => {
        res.type('html').send(html);
    })
```


Finally, we tell `Express` to use these routes for the app:

```javascript
app.use('/', router);

module.exports = app;
```