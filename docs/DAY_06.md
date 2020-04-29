# Day 6

Let's remember the original estimation of routes we are trying to build:

- Provide endpoints for `users`
  - **DONE**: POST `/users/register` (sign up)
  - **DONE**: POST `/users/login` (sign in)
  - DELETE `/users/:userId` (deactivate account)
- Provide endpoints for `posts`
  - **DONE-ish**: GET `/posts` (see posts)
  - POST `/posts` (create post)
  - PATCH `/posts/:postId` (update post)
  - DELETE `/posts/:postId` (deactivate post)
- Provide endpoints for `tags`
  - **DONE**: GET `/tags` (list of all tags)
  - GET `/tags/:tagName/posts` (list of all posts with that tagname)

We will leave user deactivation and post deactivation to the end, since they will trigger a number of updates we need to do to our database, but for now let's get started with our `posts`.

For the routes we are going to write (except for GET `/tags/:tagName/posts`), we need a user to be logged in.

Let's write a utility function in `api/utils.js` that will send an error for us whenever there is no user:

```js
// api/utils.js
function requireUser(req, res, next) {
  if (!req.user) {
    next({
      name: "MissingUserError",
      message: "You must be logged in to perform this action"
    });
  }
  
  next();
}

module.exports = {
  requireUser
}
```

It's pretty straight forward, if the `req.user` hasn't been set (which means a correct auth token wasn't sent in with the request), we will send an error rather than moving on to the actual request. Otherwise, we pass on to the next thing like nothing happened.

Writing this function will allow us to reuse it in multiple places, and one way you can use it is like this:

```js
someRouter.post('/some/route', requireUser, async (req, res, next) => {

});
```

When you pass in multiple functions after the route string, it will attach each as middleware in the order they are specified.  This means that when a user posts to `/some/route`, the `requireUser` middleware will be called first, and then if `next()` is called cleanly from `requireUser`, our anonymous middleware function will be called.

## Writing `POST /api/posts`

Now, back to `api/posts.js`:

```js
// api/posts.js

// near the top
const { requireUser } = require('./utils');

postsRouter.post('/', requireUser, async (req, res, next) => {
  res.send({ message: 'under construction' });
});
```
This is a start. We can test it out immediately:

```bash
curl http://localhost:3000/api/posts -X POST
# {"name":"MissingUserError","message":"You must be logged in to perform this action"}

curl http://localhost:3000/api/users/login -H "Content-Type: application/json" -X POST -d '{"username": "albert", "password": "bertie99"}'

# {"message":"you're logged in!","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwNjk3OTcsImV4cCI6MTU4ODY3NDU5N30.xwsxdTFC38eZFTS8h5RMsEgAmz1vw-ZizTka0d-jaYA"}

curl http://localhost:3000/api/posts -X POST -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwNjk3OTcsImV4cCI6MTU4ODY3NDU5N30.xwsxdTFC38eZFTS8h5RMsEgAmz1vw-ZizTka0d-jaYA'

# {"message":"under construction"}
```

Great! Our `requireUser` is doing the trick, and if we make it inside our anonymous function we know, without a doubt, that we have a valid user assigned to `req.user`.

Go back in and destructure the function `createPost` from our db require statement near the top. 

Remember, the function looks like this:

```js
// in db/index.js

async function createPost({
  authorId,
  title,
  content,
  tags = []
}) {
  try {
    const { rows: [ post ] } = await client.query(`
      INSERT INTO posts("authorId", title, content) 
      VALUES($1, $2, $3)
      RETURNING *;
    `, [authorId, title, content]);

    const tagList = await createTags(tags);

    return await addTagsToPost(post.id, tagList);
  } catch (error) {
    throw error;
  }
}
```

When we call this it is expecting it to be called with an object, with keys `authorId`, `title`, `content`, and `tags`. It is also expecting `tags` to be an array.

This is the contract between the server and the database, as dictated by the code we wrote for the database. However, we might want a simpler contract between our front-end and our API. There's no reason for it to be the same. We can improve the front-end coding experience by taking a bit of a hit in our server-side code.

Let's say that a valid request from the front end to `POST /api/posts` should have `title`, `content`, and `tags` all as strings. We can break the `tags` string into separate tags with a little work:

```js
const { requireUser } = require('./utils');

postsRouter.post('/', requireUser, async (req, res, next) => {
  const { title, content, tags = "" } = req.body;

  const tagArr = tags.trim().split(/\s+/)
  const postData = {};

  // only send the tags if there are some to send
  if (tagArr.length) {
    postData.tags = tagArr;
  }

  try {
    // add authorId, title, content to postData object
    // const post = await createPost(postData);
    // this will create the post and the tags for us
    // if the post comes back, res.send({ post });
    // otherwise, next an appropriate error object 
  } catch ({ name, message }) {
    next({ name, message });
  }
});
```

What is happening up above with `tags.trim().split(/\s+/)` is pretty neat. First the call to `trim()` removes any spaces in the front or back, and then split will turn the string into an array, splitting over any number of spaces. If the front-end sends us `"       #happy       #bloated #full"`, then `tagArr` will be equal to `["#happy", "#boated", "#full"]`.

I want you to write the rest of the try block using the code I provided. Once you're done you can test it out with a curl. The only two tricks are that you need to figure out what to pass in for key `authorId` (hint, we have access to the current user), and what to pass in for the key `tags`.

Remember to make sure you're using the correct authentication token.

```bash
curl http://localhost:3000/api/users/login -H "Content-Type: application/json" -X POST -d '{"username": "albert", "password": "bertie99"}'

# {"message":"you're logged in!","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwNjk3OTcsImV4cCI6MTU4ODY3NDU5N30.xwsxdTFC38eZFTS8h5RMsEgAmz1vw-ZizTka0d-jaYA"}

# Correctly formed request
curl http://localhost:3000/api/posts -X POST -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwNjk3OTcsImV4cCI6MTU4ODY3NDU5N30.xwsxdTFC38eZFTS8h5RMsEgAmz1vw-ZizTka0d-jaYA' -H 'Content-Type: application/json' -d '{"title": "test post", "content": "how is this?", "tags": " #once #twice    #happy"}'

# {"id":4,"title":"test post","content":"how is this?","active":true,"tags":[{"id":1,"name":"#happy"},{"id":11,"name":"#once"},{"id":12,"name":"#twice"}],"author":{"id":1,"username":"albert","name":"Newname Sogood","location":"Lesterville, KY"}}

# Missing tags
curl http://localhost:3000/api/posts -X POST -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwNjk3OTcsImV4cCI6MTU4ODY3NDU5N30.xwsxdTFC38eZFTS8h5RMsEgAmz1vw-ZizTka0d-jaYA' -H 'Content-Type: application/json' -d '{"title": "I still do not like tags", "content": "CMON! why do people use them?"}'

# {"id":7,"title":"I still do not like tags","content":"CMON! why do people use them?","active":true,"tags":[{"id":14,"name":""}],"author":{"id":1,"username":"albert","name":"Newname Sogood","location":"Lesterville, KY"}}

# Missing title or content
curl http://localhost:3000/api/posts -X POST -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwNjk3OTcsImV4cCI6MTU4ODY3NDU5N30.xwsxdTFC38eZFTS8h5RMsEgAmz1vw-ZizTka0d-jaYA' -H 'Content-Type: application/json' -d '{"title": "I am quite frustrated"}'

# {"name":"error","message":"null value in column \"content\" violates not-null constraint"}
```

## Updating a Post

### Params

Updating a post is very similar to creating a post, with a few changes. The first change is how we form the route: `PATCH /api/posts/:postId`. The verb `PATCH` tells a server that we wish to update some data. It's not magic, we have to write the handler for it, but it's an agreed upon standard.

The second is that `:postId`. Express uses strings like that to transform whatever is in that position to a variable we can get off the request object:

```js
server.get('/background/:color', (req, res, next) => {
  res.send(`
    <body style="background: ${ req.params.color };">
      <h1>Hello World</h1>
    </body>
  `);
});
```

This sets up a route we can hit: `/background/:color`, and whatever we put in the second spot will be set on `req.params.color`.

```bash
curl localhost:3000/background/blue

# <body style="background: blue;">
#   <h1>Hello World</h1>
# </body>

curl localhost:3000/background/magenta

# <body style="background: magenta;">
#   <h1>Hello World</h1>
# </body>
```

So we could set up a route like this:

```js
server.get('/add/:first/to/:second', (req, res, next) => {
  res.send(`<h1>${ req.params.first } + ${ req.params.second } = ${
    Number(req.params.first) + Number(req.params.second)
   }</h1>`);
});
```

And hit it:

```bash
curl localhost:3000/add/3/to/11

# <h1>3 + 11 = 14</h1>
```

There is one problem with routes like this. The parameters act exactly like wildcards. So, for example, if we went to `/add/hello/to/goodbye`, then "hello" and "goodbye" would be added.

One trap that often happens to new developers is something like this:

```js
server.get('/posts/:postId', showSinglePostPage);
server.get('/posts/edit', showEditPage);
```

What would happen when we go to `/posts/edit`? The answer is that we only make it to `showSinglePostPage`, because the wildcard `:postId` matches _anything_, including the word `edit`. That route matches, so we call `showSinglePostPage`.

To that end, wildcard routes are usually added later in a router, or at least after anything else they might match.

### Let's write `PATCH /api/posts/:postId`

Ok, so with the knowledge of how to read a param from the request, we can set up a route that uses it. Up near the top, please pull `updatePost` out of our `db` require statement so we can use it!

We are going to expect the body to have the title, or content, or maybe tags. We won't allow the user to change the `id` or the `authorId` (that would be silly for our app).

We may not be passed all fields, so we should gingerly build the object, checking to see which ones are passed in.

Lastly, we should ensure that the post we are trying to update is actually owned by the the user trying to update it. While our front-end might be good at preventing this problem, our back-end should also ensure it is not possible either.

```js
// in api/posts.js

postsRouter.patch('/:postId', requireUser, async (req, res, next) => {
  const { postId } = req.params;
  const { title, content, tags } = req.body;

  const updateFields = {};

  if (tags && tags.length > 0) {
    updateFields.tags = tags.trim().split(/\s+/);
  }

  if (title) {
    updateFields.title = title;
  }

  if (content) {
    updateFields.content = content;
  }

  try {
    const originalPost = await getPostById(postId);

    if (originalPost.author.id === req.user.id) {
      const updatedPost = await updatePost(postId, updateFields);
      res.send({ post: updatedPost })
    } else {
      next({
        name: 'UnauthorizedUserError',
        message: 'You cannot update a post that is not yours'
      })
    }
  } catch ({ name, message }) {
    next({ name, message });
  }
});
```

And we can test it out the following way (remember to use your bearer token):

```bash
curl http://localhost:3000/api/posts/1 -X PATCH -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwNjk3OTcsImV4cCI6MTU4ODY3NDU5N30.xwsxdTFC38eZFTS8h5RMsEgAmz1vw-ZizTka0d-jaYA' -H 'Content-Type: application/json' -d '{"title": "updating my old stuff", "tags": "#oldisnewagain"}'
```

## Writing `GET /api/tags/:tagName/posts`

This is one for you to write, but there is one thing we will need to do to test it, and one thing we'll need to remember when we write the front end for it.

### Escaping our Characters

Our tags, so far, have looked like common internet tags: a hash symbol (`#`) and a string. When we fetch to this endpoint, we might be tempted to to this:

```js
fetch(`/api/tags/${ tag }/posts`)
  .then()
```

Where `tag` is one of our tags like `#happy`, the url will look like this: `/api/tags/#happy/posts`. 

You might remember, way back when, we implemented a basic hash router. When there is a [hash string](https://developer.mozilla.org/en-US/docs/Web/API/URL/hash) in a url, everything after the `#` gets treated as a fragment, and if you were to try to hit a route setup as `/api/tags/:tagName/posts` with a hash symbol in the place of `:tagName`, it simply won't connect.

This is where [encodeUriComponent](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent) will come into play. This is a note for us in the future, but for now when we test out our route we will just replace the `#` symbol with `%23`, something known as [percent encoding](https://en.wikipedia.org/wiki/Percent-encoding).

### The Route

You should work with the `tagsRouter` in `api/tags.js`. And flesh out this function. Don't forget to import the appropriate function from our database layer.

```js
tagsRouter.get('/:tagName/posts', async (req, res, next) => {
  // read the tagname from the params
  try {
    // use our method to get posts by tag name from the db
    // send out an object to the client { posts: // the posts }
  } catch ({ name, message }) {
    // forward the name and message to the error handler
  }
});
```

You can test this out like this:

```bash
curl http://localhost:3000/api/tags/%23happy/posts
```

## Deactivating Posts

Let's start our journey into deactivation at the post level. We will need to do two things:

1. Hook up the route that will let us deactivate the post.
2. Possibly update existing methods/routes that return posts to the client depending on if they should have access to deactivated posts (or not).

For now we can start with number 1.

### Setting up `DELETE /posts/:postId`

Because we've set up an `active` column in our post table, and (on creation) set it to `true` by default, we can simply update a post to have `active: false` (not changing anything else) to delete it.

```js
postsRouter.delete('/:postId', requireUser, async (req, res, next) => {
  try {
    const post = await getPostById(req.params.postId);

    if (post && post.author.id === req.user.id) {
      const updatedPost = await updatePost(post.id, { active: false });

      res.send({ post: updatedPost });
    } else {
      // if there was a post, throw UnauthorizedUserError, otherwise throw PostNotFoundError
      next(post ? { 
        name: "UnauthorizedUserError",
        message: "You cannot delete a post which is not yours"
      } : {
        name: "PostNotFoundError",
        message: "That post does not exist"
      });
    }

  } catch ({ name, message }) {
    next({ name, message })
  }
});
```

And we can test it out like this:

```bash
curl http://localhost:3000/api/posts/1 -X DELETE -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwNjk3OTcsImV4cCI6MTU4ODY3NDU5N30.xwsxdTFC38eZFTS8h5RMsEgAmz1vw-ZizTka0d-jaYA'

# {"post":{"id":1,"title":"New Title","content":"I really do not want to complain","active":false,"tags":[{"id":11,"name":"#oldisnewalways"}],"author":{"id":1,"username":"albert","name":"Newname Sogood","location":"Lesterville, KY"}}}
```

You should also try to delete a post not owned by the user, as well as a post which does not exist.

### Unexpected Problems

If you try the last, you might notice a new error pop up... we hadn't anticipated this earlier, but our code might try to call `getPostById` with an invalid `id`. We can fix that method by adding the following:

```js
// in db/index.js

async function getPostById(postId) {
  try {
    const { rows: [ post ]  } = await client.query(`
      SELECT *
      FROM posts
      WHERE id=$1;
    `, [postId]);

    // THIS IS NEW
    if (!post) {
      throw {
        name: "PostNotFoundError",
        message: "Could not find a post with that postId"
      };
    }
    // NEWNESS ENDS HERE

    const { rows: tags } = await client.query(`
      SELECT tags.*
      FROM tags
      JOIN post_tags ON tags.id=post_tags."tagId"
      WHERE post_tags."postId"=$1;
    `, [postId])

    const { rows: [author] } = await client.query(`
      SELECT id, username, name, location
      FROM users
      WHERE id=$1;
    `, [post.authorId])

    post.tags = tags;
    post.author = author;

    delete post.authorId;

    return post;
  } catch (error) {
    throw error;
  }
}
```

Now if we try to get a post which doesn't exist, we throw the appropriate error early, and don't try to attach tags and author info for a post which doesn't exist.

It's ok to have new problems crop up while you are coding out new functionality, by the way. The alternative is to spend countless hours planning out your functionality only to have to scrap it at a moments notice.

## Updating Other Methods

Now when we query all posts, we are going to get a combination of active and inactive posts. We need to make a decision on what to do with these methods.

After we deleted a post:

```bash
curl http://localhost:3000/api/posts

# {"posts":[{"id":2,"title":"How does this work?","content":"Seriously, does this even do anything?","active":true,"tags":[{"id":2,"name":"#youcandoanything"},{"id":9,"name":"#redfish"},{"id":10,"name":"#bluefish"}],"author":{"id":2,"username":"sandra","name":"Just Sandra","location":"Ain't tellin'"}},{"id":3,"title":"Living the Glam Life","content":"Do you even? I swear that half of you are posing.","active":true,"tags":[{"id":1,"name":"#happy"},{"id":2,"name":"#youcandoanything"},{"id":7,"name":"#canmandoeverything"}],"author":{"id":3,"username":"glamgal","name":"Joshua","location":"Upper East Side"}},{"id":1,"title":"New Title","content":"I really do not want to complain","active":false,"tags":[{"id":11,"name":"#oldisnewalways"}],"author":{"id":1,"username":"albert","name":"Newname Sogood","location":"Lesterville, KY"}}]}
```

Look at the last one...

```js
{
  "id": 1,
  "title": "New Title",
  "content": "I really do not want to complain",
  "active": false,
  "tags": [{"id":11,"name":"#oldisnewalways"}],
  "author": {"id":1,"username":"albert","name":"Newname Sogood","location":"Lesterville, KY"}
}
```

Active is now false, but do we want inactive posts when we get all posts?

We have to places we could change this: at the Database Layer, or at the API Layer.

I would like to change it at the API layer. Let's leave the database methods alone, and filter out any posts before we return them. Something like this:

```js
// in api/posts.js

postsRouter.get('/', async (req, res) => {
  try {
    const allPosts = await getAllPosts();

    const posts = allPosts.filter(post => {
      // keep a post if it is either active, or if it belongs to the current user
    });
  
    res.send({
      posts
    });
  } catch ({ name, message }) {
    next({ name, message });
  }
});
```

How would we write that filter function? Remember that a filter function should return something truthy if we want to keep the object, or something falsy if we don't.

```js
const posts = allPosts.filter(post => {
  // the post is active, doesn't matter who it belongs to
  if (post.active) {
    return true;
  }

  // the post is not active, but it belogs to the current user
  if (req.user && post.author.id === req.user.id) {
    return true;
  }

  // none of the above are true
  return false;
});
```

So we have to do the `req.user && ....` trick, because if `req.user` isn't defined, we can't call `.id` on it.

When you have a number of if statements all returning true or false, it's possible to clean up the code:

```js
if (someConditional) {
  return true;
} else {
  return false;
}

// is the same as:

return someConditional;
```

So another way to write that filter is this:

```js
const posts = allPosts.filter(post => {
  return post.active || (req.user && post.author.id === req.user.id);
});
```

That isn't bad, it fits on one line, and might be readable. If we have to make changes, however, it will start to buckle under the weight of its own cleverness.

Don't let [playing code golf](https://en.wikipedia.org/wiki/Code_golf) get in the way of writing maintainable code. 

### Test it out

```bash
curl http://localhost:3000/api/posts

# should not show the inactive post

curl http://localhost:3000/api/posts -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhbGJlcnQiLCJpYXQiOjE1ODgwNjk3OTcsImV4cCI6MTU4ODY3NDU5N30.xwsxdTFC38eZFTS8h5RMsEgAmz1vw-ZizTka0d-jaYA'

# should show active posts, _plus_ inactive posts for that user
```

### Update `GET /api/tags/:tagName/posts`

You should now update this method to filter out any posts which are **both** inactive and not owned by the current user.

Make sure to test it out by curling both `http://localhost:3000/api/tags/%23sometagname` and also by passing a bearer token for a user with an inactive post.

## Wrap up for now

Make sure you're happy with your code, commit it, and send it up to GitHub.

We've now made almost all routes we set out to make, and we have a solid API complete with user authentication.

## Stretch Goals

Ok, so at this point you're set with almost all things, but here are some you could implement knowing what you know now:

### Add `DELETE /api/users/:userId`

This one, like deleting a post, requires you to:

- verify the user exists at all
- verify the logged in user is the same as the one being deleted
- update the user to have `active` set to false

Once a user is deactivated you'll have to make decisions similar to the ones we've made previously with deactivated posts:

- `GET /api/posts` probably should **now** also filter out posts by not-active authors (`post.author.active` is available for this)... but maybe not if the current user is the inactive author
- `GET /api/tags/:tagName/posts` should do the same
- Any creation, updating, or deleting of posts should not work for an inactive user. You could create a new method `requireActiveUser` which is like `requireUser`, but also checks to see if the user is active. Then you could add that as middleware to any route that needs an active user

We don't need a separate route to activate users, we could just use `PATCH /api/users/:userId` to set `active` to true. This would allow our front-end to give users a chance to toggle active status.