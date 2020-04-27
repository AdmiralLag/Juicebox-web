# Day 4

Today we are going to start building our web server. How exciting!

## Getting Started

### Install Packages

Navigate back to your project folder, and from the shell install the following packages:

```bash
npm install express
```

`Express` is our web server. It gives us many options, 

### Start a web server

Here is some boilerplate to get started:

```js
// inside index.js
const PORT = 3000;
const express = require('express');
const server = express();

server.listen(PORT, () => {
  console.log('The server is up on port', PORT)
});
```

And update your `package.json` to include two new scripts:

```js
{
  // other stuff
  "scripts": {
    "seed:dev": "nodemon ./db/seed.js",
    "start": "node index.js", // new
    "start:dev": "nodemon index.js" // new
  },
  // more stuff
}
```

Lastly, from the terminal, run `npm run start:dev` and you should see this:

```bash
> nodemon index.js

[nodemon] 2.0.3
[nodemon] to restart at any time, enter `rs`
[nodemon] watching path(s): *.*
[nodemon] watching extensions: js,mjs,json
[nodemon] starting `node index.js`
The server is up on port 3000
```

## Middleware

[Express middleware](https://expressjs.com/en/guide/using-middleware.html) is, at the most basic level, any function that can run between a request coming in and a response going out.

### Our First Middleware

We will write functions and set up the conditions upon which they should be run. Here's a simple example:

```js
server.use((req, res, next) => {
  console.log("<____Body Logger START____>");
  console.log(req.body);
  console.log("<_____Body Logger END_____>");

  next();
});
```

The call on `server.use` tells the server to *always* call this function.

The server will pass in:

- the request object (built from the client's request)
- the response object (which has methods to build and send back a response)
- the `next` function, which will move forward to the next matching middleware

Our function will run on every request, will log out 3 lines including one built from the request object's `body` property (which may be undefined), and finally calls `next` in order to move on to another piece of middleware.

Test it out by opening `http://localhost:3000` in your browser while your server is running and looking at the terminal.

### And now another

When we attach middleware, we can specify a few things:

- The method: `get`, `post`, `patch`, `put`, and `delete`, or method agnostic (`use`)
- An optional request path that must be matched, e.g. `/api/users`, or even with a placeholder `/api/users/:userId`
- A function with either three or four parameters
  - three parameter needs `request`, `response`, and `next` *in that order*
  - four parameter needs `error`, `request`, `response` and `next`, *in that order*
  - four parameter functions are considered error handling middleware (which is why the error is prioritized)

The order which we attach the middleware matters. Matches are called one at a time, first in first out.

Look at the example below. Assume someone made a GET request to /api:

```js
app.use('/api', (req, res, next) => {
  console.log("A request was made to /api");
  next();
});

app.get('/api', (req, res, next) => {
  console.log("A get request was made to /api");
  res.send({ message: "success" });
});
```

What would happen? What would happen in the same scenario, below?

```js
app.get('/api', (req, res, next) => {
  console.log("A get request was made to /api");
  res.send({ message: "success" });
});
  
app.use('/api', (req, res, next) => {
  console.log("A request was made to /api");
  next();
});
```

In the first scenario `app.use` would fire (it likes all types of requests), log, and then call next. Since `app.get` matches the same path, it would then fire, log, and send back a JSON response.

In the second, `app.get` would fire (it matches the request precisely), log, and then send back a JSON response. Since it doesn't call `next()` (and moreover, shouldn't since it is sending back a response), we never move forward to `app.use` and so that second log won't happen.

However, if a `POST` request was made to `/api`, the `app.get` middleware would be skipped over, and the `app.use` would go off, log, and move on to the next match.

## Collecting Common Middleware

Often we have a collection of common routes. For our app, we will define the following routes:

```
POST /api/users/register
POST /api/users/login
DELETE /api/users/:id

GET /api/posts
POST /api/posts
PATCH /api/posts/:id
DELETE /api/posts/:id

GET /api/tags
GET /api/tags/:tagName/posts
```

There is one common root path ('/api'), as well as three sets of subroutes.

One way to handle this would be like this:

```js
server.post('/api/users/register', () => {});
server.post('/api/users/login', () => {});
server.delete('/api/users/:id', () => {});
```

### Our first router

Express lets us attach a collection of routes in a structured way.

First, create a new folder called `api`, and in it create files `index.js`, `users.js`, `posts.js`, and `tags.js`.

Let's start with `users.js`:

```js
// api/users.js
const express = require('express');
const usersRouter = express.Router();

usersRouter.use((req, res, next) => {
  console.log("A request is being made to /users");

  res.send({ message: 'hello from /users!' });
});

module.exports = usersRouter;
```

The `express` object is useful for more than creating a server. Here we use the `Router` function to create a new router, and then export it from the script.

Now, inside the new `index.js` we can require and attach it to an `apiRouter`:

```js
// api/index.js
const express = require('express');
const apiRouter = express.Router();

const usersRouter = require('./users');
apiRouter.use('/users', usersRouter);

module.exports = apiRouter;
```

And lastly in our main `index.js`:

```js
// stuff above here

const apiRouter = require('./api');
server.use('/api', apiRouter);

// stuff below here
```

And we have created a tree of routes:

- Any time a request is made to a path starting with `/api` the `apiRouter` will be held responsible for making decisions, calling middleware, etc.
- `apiRouter` will match paths now with the `/api` portion removed
- This means that if we hit `/api/users`, `apiRouter` will try to match `/users` (which it can), and it will then pass on the responsibility to the `usersRouter`
- Finally `usersRouter` will try to match (now with `/api/users` removed from the original matching path), and fire any middleware.

We've set up a sensible directory structure, kept our concerns small without being too small, and set ourselves up for success.

Try going to `http://localhost:3000/api/users` and see what happens. Did you get some JSON?

Look in the console, do you see a message?

### Add some useful middleware

We currently have the body logging middleware we created, and we also have a log inside of `usersRouter`, but we would like two things:

1. To turn incoming request bodies into useful objects. `body-parser` is bundled with express, so we don't need to install a new package to use it.
2. To log out each incoming request without us having to write a log in each route. `morgan` is a good package which achieves this:

    ```bash
    npm i morgan
    ```

Then, inside our server, near the top (before our body-logging middleware):

```js
const bodyParser = require('body-parser');
server.use(bodyParser.json());

const morgan = require('morgan');
server.use(morgan('dev'));
```

The first, `bodyParser.json()`, is a function which will read incoming JSON from requests. The request's header has to be `Content-Type: application/json`, but we get the benefit of being able to send objects easily to our server.

The second, `morgan('dev')`, is a function which logs out the incoming requests, like so:

```
GET /api/users 304 19.825 ms - -
```

We get the method, the route, the HTTP response code (here, 304), and how long it took to form.

### An Example Route

Inside our `usersRouter`, let's add a simple route:

```js
// api/users.js
const express = require('express');
const usersRouter = express.Router();

usersRouter.use((req, res, next) => {
  console.log("A request is being made to /users");
  
  next(); // THIS IS DIFFERENT
});

usersRouter.get('/', (req, res) => {
  res.send({
    users: []
  });
});

module.exports = usersRouter;
```

1. That middleware will fire whenever a GET request is made to `/api/users`
2. It will send back a simple object, with an empty array.

Wouldn't it be neat if we could actually get the users from the database, and... send them back? 

We can! (surprise!) The whole point behind building the data layer is to be able to use it when a user request comes in...

```js
// api/users.js

// NEW
const { getAllUsers } = require('../db');

// UPDATE
usersRouter.get('/', async (req, res) => {
  const users = await getAllUsers();

  res.send({
    users
  });
});
```

Now when a request comes in, we first ask the database for the data we want, then send it back to the user.

Try to hit `http://localhost:3000/api/users`. What happens? Why does our application freeze?

We never connected our client! Back in our main `index.js`, we should connect to the client right before starting up our server:

```js
// index.js

const { client } = require('./db');
client.connect();

server.listen(PORT, () => {
  // old stuff
});
```

Go back to the route, and you should get a JSON response with our three users! Hurray!

### Write more routes

#### `GET /api/posts`

Right now, another safe route to write is `GET /api/posts`. Getting all the posts does not require a user to be logged in, so we need not worry about that.

Go to `postsRouter.js`, first create a new router (like we did in `usersRouter.js`). Make sure to export it at the end.

Then, add your own middleware to run when the user makes a `GET` request to `/api/posts`. When they do, call `getAllPosts` from our database (don't forget to require it), and return the result.

It would be nice if the result was an object like this:

```js
{
  "posts": []
}
```

Then, once you've written the code there, go back into `api/index.js`, and import and attach the `postsRouter` to our `apiRouter`.

Test it out, when you're done, by going to `http://localhost:3000/api/posts`

#### `GET /api/tags`

Do the same as above, but now with `tags`.

#### Test it out

Navigate to `http://localhost:3000/api/posts` and `http://localhost:3000/api/tags` to make sure we can get the data from our seeds.

### Wrap Up

Today we laid the foundation upon which we will build out the rest of our web server. 

Make sure to commit your code and push it up to your remote.