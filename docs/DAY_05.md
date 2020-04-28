# Day 5

We have two major things left to do:

- Enable User Authentication through JWT
- Finish the required routes

Today we will implement JSON Web Tokens, and start the routes.

## JSON Web Tokens

### Installing the package and playing with it

To get started we need to install one new packages:

```bash
npm i jsonwebtoken
```

We need to use this in two ways:

1. To create a JSON web token using a combination of a server-side-secret and our user-supplied data (we will use the username)
2. To decrypt the JSON web token that the user sends to us during our requests.

Drop into the node repl (type `node` on your command line):

```js
const token = jwt.sign({ id: 3, username: 'joshua' }, 'server secret');

token; // 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Impvc2h1YSIsImlhdCI6MTU4ODAyNDQwNn0.sKuQjJRrTjmr0RiDqEPJQcTliB9oMACbJmoymkjph3Q'

const recoveredData = jwt.verify(token, 'server secret');

recoveredData; // { id: 3, username: 'joshua', iat: 1588024406 }
```

The key `iat` refers to when the token was issued. Like all good times it is measured in milliseconds since [the epoch](https://en.wikipedia.org/wiki/Unix_time).

You are able to add a third parameter to `jwt.sign` like this:

```js
const token = jwt.sign({ id: 3, username: 'joshua' }, 'server secret', { expiresIn: '1h' });

token; // 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Impvc2h1YSIsImlhdCI6MTU4ODAyNDkwMSwiZXhwIjoxNTg4MDI4NTAxfQ.LGqAMv7Bc7xKKHiQp8m4bpqR53h5dJBOZ4Kv2b9qmqY'

const recoveredData = jwt.verify(token, 'server secret');

recoveredData; // { id: 3, username: 'joshua', iat: 1588024901, exp: 1588028501 }

// wait 1 hour:

jwt.verify(token, 'server secret');

// Uncaught TokenExpiredError: jwt expired {
//   name: 'TokenExpiredError',
//   message: 'jwt expired',
//   expiredAt: 2020-04-27T21:58:57.000Z
// }
```

Cool, so there's not only a way to make a token, but also set a finite amount of time for it to be valid. This is a way we could force our users to reauthenticate every so often.

For now, we'll choose to let our tokens last 1 week. If we need to, we can change that in the future.

### Building some infrastructure

We want to create some middleware that will look for the correct payload from our requests. 

#### The user request

We will require the front end to make requests that look like this:

```js
fetch('our api url', {
  method: 'SOME_METHOD',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer HOLYMOLEYTHISTOKENISHUGE'
  },
  body: JSON.stringify({})
})
```

We need the `Content-Type` set so that our `bodyParser` module will be able to read off anything we need from the user (like form data).

We need the `Authorization` set so that we can read off that Bearer token.  It will look something like this:

```js
server.use(async (req, res, next) => {
  const prefix = 'Bearer '
  const auth = req.headers['Authorization'];

  if (!auth) {
    next(); // don't set req.user, no token was passed in
  }


  if (auth.startsWith(prefix)) {
    // recover the token
    const token = auth.slice(prefix.length);
    try {
      // recover the data
      const { id } = jwt.verify(data, 'secret message');

      // get the user from the database
      const user = await getUserById(id);
      // note: this might be a user or it might be null depending on if it exists

      // attach the user and move on
      req.user = user;

      next();
    } catch (error) {
      // there are a few types of errors here
    }
  }
})
```

This functionality should be protecting our api, so let's go add it to `api/index.js`. But first...

## Protecting our Secrets

As we saw in our first node project, we shouldn't publish our secrets. I'll walk us through the steps one more time:

### Create a .env file

In our project's root folder, create a new file called `.env` (the dot is important):

```js
touch .env
```

Then, edit the file, and add a key/value pair like so:

```
JWT_SECRET="don't tell a soul"
```

Lastly, protect our secrets by adding the file to our `.gitignore`:

```
node_modules
.env
```

### Rehydrate our Secret

Install the package `dotenv`:

```bash
npm i dotenv
```

And at the very beginning of our `index.js`:

```js
require('dotenv').config();

// remove this once you confirm it works
console.log(process.env.JWT_SECRET);
// like, seriously. go delete that!

// EVERYTHING ELSE
```

Start back up the server and you should see your secret on a log line. Make sure you delete that log once you see the secret on the command line.

## Working inside `api/index.js`

Since this operation is relatively complex at first glance, I am happy to provide you with working code.

```js
// Before we start attaching our routers

const jwt = require('jsonwebtoken');
const { getUserById } = require('../db');
const { JWT_SECRET } = process.env;

// set `req.user` if possible
apiRouter.use(async (req, res, next) => {
  const prefix = 'Bearer ';
  const auth = req.header('Authorization');
  
  if (!auth) { // nothing to see here
    next();
  } else if (auth.startsWith(prefix)) {
    const token = auth.slice(prefix.length);

    try {
      const { id } = jwt.verify(token, JWT_SECRET);

      if (id) {
        req.user = await getUserById(id);
        next();
      }
    } catch ({ name, message }) {
      next({ name, message });
    }
  } else {
    next({
      name: 'AuthorizationHeaderError',
      message: `Authorization token must start with ${ prefix }`
    });
  }
});

// Attach routers below here
```

You can see by the `if/else if/else` we have three possibilities with **every** request to `/api`:

1. **IF**: The Authorization header wasn't set. This might happen with registration or login, or when the browser doesn't have a saved token. Regardless of why, there is no way we can set a user if their data isn't passed to us.
2. **ELSE IF**: It was set, and begins with `Bearer` followed by a space. If so, we'll read the token and try to decrypt it.
  a. On successful `verify`, try to read the user from the database
  b. A failed `verify` throws an error, which we catch in the catch block. We read the `name` and `message` on the error and pass it to `next()`.
3. **ELSE**: A user set the header, but it wasn't formed correctly. We send a `name` and `message` to `next()`

Ok... so in one case we might add a key to the `req` object, and in two of the cases we might pass an error object to `next`.

## Dealing with Errors

Express's normal way of dealing with an error is serving up an error page with the error stringified.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Error</title>
</head>
<body>
<pre>Cannot GET /what</pre>
</body>
</html>
```

Since we are using our server as an API, it would be more useful to us for any errors coming out of a request to `/api` (and its children) to be a JSON error object.

So, at the bottom of `api/index.js` (after all routers are attached) we can add a simple error handler:

```js
// api/index.js

// all routers attached ABOVE here
apiRouter.use((error, req, res, next) => {
  res.send(error);
});

module.exports = apiRouter;
```

Now any time middleware that the `apiRouter` might be the parent router for calls `next` with an object (rather than just `next()`), we will skip straight to the error handling middleware and send back the object to the front-end.

### Testing it out

Start two terminal windows, in one run the server with `npm run start:dev`, and in the other, try these commands:

```bash
curl http://localhost:3000

curl http://localhost:3000/api

curl http://localhost:3000/api -H 'Authorization: Bearer xyz'

curl http://localhost:3000/api -H 'Authorization: The Bears xyz'

curl http://localhost:3000/api/posts
```

And you'll see different responses. For the two `curl` commands with headers, you should get both types of errors (`AuthorizationHeaderError` and `JsonWebTokenError`).

## Giving Them a Token

So there are two instances where we want to give our users a token... when they register and when they login.

### Login

Let's start with logging in, since it's a _bit_ easier.

To test your code as you write it, you can use this curl command:

```bash
curl http://localhost:3000/api/users/login -H "Content-Type: application/json" -X POST -d '{"username": "albert", "password": "bertie99"}' 
```

Of course, use a valid username & password combo from your seeds. But this matches mine.

Then, we need to actually set up that route, so let's go to `api/users.js` and create it:

```js
// api/users.js

usersRouter.post('/login', async (req, res, next) => {
  console.log(req.body);
  res.end();
});
```

If you execute that `curl` command, you'll see your server log out `{ username: 'albert', password: 'bertie99' }`. This is a great start: `bodyParser.json` is reading off of our request, and attaching an object to our request object.

Now we should try to verify that our user exists, and if so, send back a JWT to the front-end.

Looking over our database calls, we have one to look up a user by `id`, but not by `username`. Maybe we should go create that:

### Hop over to `db/index.js`

Add this method, and make sure to export it:

```js
// db/index.js

async function getUserByUsername(username) {
  try {
    const { rows: [user] } = await client.query(`
      SELECT *
      FROM users
      WHERE username=$1
    `, [username]);

    return user;
  } catch (error) {
    throw error;
  }
}
```

Then, back inside `api/users.js`, we can do something similar to this:

```js
usersRouter.post('/login', async (req, res, next) => {
  const { username, password } = req.body;

  // request must have both
  if (!username || !password) {
    next({
      name: "MissingCredentialsError",
      message: "Please supply both a username and password"
    });
  }

  try {
    const user = await getUserByUsername(username);

    if (user && user.password == password) {
      // create token & return to user
      res.send({ message: "you're logged in!" });
    } else {
      next({ 
        name: 'IncorrectCredentialsError', 
        message: 'Username or password is incorrect'
      });
    }
  } catch(error) {
    console.log(error);
    next(error);
  }
});
```

You'll see there are some missing pieces, but for now test it out -- use the correct password and the incorrect password, also try to call it with missing information.

We've covered our bases, but we're not sending anything back that's interesting when a user logs in. You should:

1. Require the `jsonwebtoken` package, store it in a constant `jwt`
2. Sign an object that has both the `id` and `username` from the `user` object with the secret in `process.env.JWT_SECRET`
3. Add a key of token, with the token returned from step 2, to the object passed to `res.send()`

When you successfully make a call to the server, you should get a response like this:

```bash
curl http://localhost:3000/api/users/login -H "Content-Type: application/json" -X POST -d '{"username": "albert", "password": "bertie99"}'

{"message":"you're logged in!","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwMzU3MDN9.YdIMbfMPHoYoXUGxeo1fQOhEpKB_esw_pEzqSUXdRVg"}
```

That token is the JWT that we will pass in when we want to hit another route.

### Let's test out our token

Back in `/api/index.js`, after the middleware we attached to read the JWT, add one more middlware (before any other routes are attached:

```js
// api/index.js

// JWT middware above here

apiRouter.use((req, res, next) => {
  if (req.user) {
    console.log("User is set:", req.user);
  }

  next();
});

// Routers below here
```

Then, take the token you got above (**don't use mine, actually use the one you get by logging in your user**), curl as below:

```bash
curl http://localhost:3000/api -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwMzU3MDN9.YdIMbfMPHoYoXUGxeo1fQOhEpKB_esw_pEzqSUXdRVg'
```

And you should see this:

```bash
User is set: {
  id: 1,
  username: 'albert',
  name: 'Newname Sogood',
  location: 'Lesterville, KY',
  active: true,
  posts: [
    {
      id: 1,
      title: 'New Title',
      content: 'Updated Content',
      active: true,
      tags: [Array],
      author: [Object]
    }
  ]
}
```

Hooray! We can consider users to be logged in when `req.user` is set in all of our routes.

## Create Users

When a `POST` request comes in to `/api/users/register`, we need to read off the 4 fields, and create a new user.

First, we should check to see if that username is already taken, and if so, pass `next` a reasonable error.

If not, try to create the user with the supplied fields.

On success, sign and return a token with the `user.id` and the `username`.

And, as usual: catch any errors from the try block, and forward them to our error handling middleware:

```js
usersRouter.post('/register', async (req, res, next) => {
  const { username, password, name, location } = req.body;

  try {
    const _user = await getUserByUsername(username);
  
    if (_user) {
      next({
        name: 'UserExistsError',
        message: 'A user by that username already exists'
      });
    }

    const user = await createUser({
      username,
      password,
      name,
      location,
    });

    const token = jwt.sign({ 
      id: user.id, 
      username
    }, process.env.JWT_SECRET, {
      expiresIn: '1w'
    });

    res.send({ 
      message: "thank you for signing up",
      token 
    });
  } catch ({ name, message }) {
    next({ name, message })
  } 
});
```

Try it out:

```bash
# missing a field
curl http://localhost:3000/api/users/register -H "Content-Type: application/json" -X POST -d '{"username": "syzygy", "password": "stars", "name": "josiah"}' 

# successful
curl http://localhost:3000/api/users/register -H "Content-Type: application/json" -X POST -d '{"username": "syzygys", "password": "stars", "name": "josiah", "location": "quebec"}'

# duplicate username
curl http://localhost:3000/api/users/register -H "Content-Type: application/json" -X POST -d '{"username": "syzygys", "password": "stars", "name": "josiah", "location": "quebec"}'
```

On success, you should get back a token. Try using that to log in with our previous route. Neat, right?

## Wrap up

What a day!

Go ahead, commit your code, and we will finish up the rest of the API over the next few days.