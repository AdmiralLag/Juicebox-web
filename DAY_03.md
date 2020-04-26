# Day 03

Goals for day 3:

- Create a new table, `tags`
  - `id` of type `SERIAL`, which is a `PRIMARY KEY`
  - `name` of type `VARCHAR`

- Similarly create a new table, `post_tags`
  - `id` of type `SERIAL` which is a `PRIMARY KEY`
  - `"postId"` (remember to use quotes around a cased column name)
  - `"tagId"` (remember to use quotes around a cased column name)
  - add a `UNIQUE` constraint on `("postId", "tagId")` to prevent the same pair from being created

- Add the following methods to `db/index.js`:
  - `findOrCreateTag({ name })` (finds a tag by name, if possible, otherwise creates and returns a new tag)
  - `setPostTags(postId, [ array of tagIds ])`

- Add some seeds and tests to `seed.js`

- Then update `getPostById` and `getPosts` to include the tags in the posts

    - Basically for getPosts - do a full lookup of posts *and* tags *and* post_tags and add the correct tags to the posts


## New Things

### Create Tables

We need two tables:

- `tags`
  - `id`, `SERIAL PRIMARY KEY`
  - `name`, `VARCHAR(255) UNIQUE NOT NULL`

- `post_tags`
  - `"postId"`, `INTEGER REFERENCES posts(id)`
  - `"tagId"`, `INTEGER REFERENCES tags(id)`
  - Add a `UNIQUE` constraint on `("postId", "tagId")`

So add those to `createTables`, in that order, in `seed.js`.

### Drop Tables

We also need to drop them in the correct order, since there are now references:

```js
async function dropTables() {
  try {
    console.log("Starting to drop tables...");

    // have to make sure to drop in correct order
    await client.query(`
      DROP TABLE IF EXISTS post_tags;
      DROP TABLE IF EXISTS tags;
      DROP TABLE IF EXISTS posts;
      DROP TABLE IF EXISTS users;
    `);

    console.log("Finished dropping tables!");
  } catch (error) {
    console.error("Error dropping tables!");
    throw error;
  }
}
```

### Tag Methods

The goal is this: a user creates a post:

```js
{
  title: "some title",
  content: "here's my blog post!",
  tags: [ "#first", "#best", "#glory-days" ]
}
```

We have a nice way to create the post itself, but the tags aren't stored in the post... they're stored in the tags table. But, still... the key takeaway is that tags are going to be coming in in batches.

#### `createTags`

We can insert multiple values at the same time, we just have to format our query a little bit differently. It'll look like this:

```SQL
INSERT INTO tags(name)
VALUES ('#tag'), ('#othertag'), ('#moretag')
ON CONFLICT (name) DO NOTHING;
```

And when we use the `node-postgres` interface, we have to replace the strings above with our placeholders, and pass the values as the second argument of query:

```SQL
INSERT INTO tags(name)
VALUES ($1), ($2), ($3)
ON CONFLICT (name) DO NOTHING;
```

We will need do make a string that looks like `($1), ($2), ($3)` and so on. It has to depend on the number of things in the tag list passed into our function.

Then, once we create the ones **not** currently in the table, we should select all the tags passed in from the table, and return them. That will help us make `post_tags`, in the step after this one.

```SQL
SELECT * FROM tags
WHERE name
IN ('#tag', '#othertag', '#moretag');
```

Or, using our interpolation:

```SQL
SELECT * FROM tags
WHERE name
IN ($1, $2, $3);
```

Again, we will need to make the right string so that the values interpolate correctly.

```js
async function createTags(tagList) {
  if (tagList.length === 0) { 
    return; 
  }

  // need something like: $1), ($2), ($3 
  const insertValues = tagList.map(
    (_, index) => `$${index + 1}`).join('), (');
  // then we can use: (${ insertValues }) in our string template

  // need something like $1, $2, $3
  const selectValues = tagList.map(
    (_, index) => `$${index + 1}`).join(', ');
  // then we can use (${ selectValues }) in our string template

  try {
    // insert the tags, doing nothing on conflict
    // returning nothing, we'll query after

    // select all tags where the name is in our taglist
    // return the rows from the query
  } catch (error) {
    throw error;
  }
}
```

#### `addTagsToPost`

This function will usually follow `createTags`, since we will only create tags while we create posts, and will need to create the elements in the join table after creating the tags.

Suppose we have a post:

```js
{
  id: 7, 
  title: 'stuff',
  content: 'more stuff'
}
```

And a tag list:

```js
[
  { id: 3, name: '#groovy' },
  { id: 12, name: '#firstfriday' }
]
```

We would need to create two `post_tags`: 

```js
[
  { "postId": 7, "tagId": 3 },
  { "postId": 7, "tagId": 12 }
]
```

These will be useful when we modify `getPosts` to include the tags, as well as when get create `getPostsWithTag` in a bit.

Here the tagList needs to be the ones returned from `createTags`, since we need the `id`, not the `name`. 

There are a _LOT_ of ways we can optimize this, but let's let a little inefficiency be the friend of keeping our code looking simple.

Inserting a single postTag, and ignoring any conflict on the pair of values looks like this:

```SQL
INSERT INTO post_tags("postId", "tagId")
VALUES (7, 3)
ON CONFLICT ("postId", "tagId") DO NOTHING;
```

So, as a start let's write `createPostTag`:

```js
async function createPostTag(postId, tagId) {
  try {
    await client.query(`
      INSERT INTO post_tags("postId", "tagId")
      VALUES ($1, $2)
      ON CONFLICT ("postId", "tagId") DO NOTHING;
    `, [postId, tagId]);
  } catch (error) {
    throw error;
  }
}
```

We can now use this multiple times in `addTagsToPost`. The function `createPostTag` is `async`, so it returns a promise. That means if we make an array of non-`await` calls, we can use them with `Promise.all`, and then await that:

```js
async function addTagsToPost(postId, tagList) {
  try {
    const createPostTagPromises = tagList.map(
      tag => createPostTag(postId, tag.id)
    );

    await Promise.all(createPostTagPromises);

    return await getPostById(postId);
  } catch (error) {
    throw error;
  }
}
```

That return is using a function which does not yet exist, let's write that, too. We are getting to the point where we want a lot more associated data with our queries. Now we want the tags, too.

We will want the post, and its tags, we can do that with two queries. First we need to get the `post` itself, then get its tags using a `JOIN` statement. We should also grab the author info using a simple query. 

Last we should add the `tags` and `author` to the `post` before returning it, as well as remove the `authorId`, since it is encapsulated in the `author` property.

```js
async function getPostById(postId) {
  try {
    const { rows: [ post ]  } = await client.query(`
      SELECT *
      FROM posts
      WHERE id=$1;
    `, [postId]);

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

This will now return something looking like this:

```js
{
  id: 1,
  title: 'First Post',
  content: 'This is my first post. I hope I love writing blogs as much as I love writing them.',
  active: true,
  tags: [ 
    { id: 1, name: '#happy' }, 
    { id: 3, name: '#youcandoanything' } 
  ],
  author: {
    id: 1,
    username: 'albert',
    name: 'Al Bert',
    location: 'Sidney, Australia'
  }
}
```

Very nice!

#### Test it out

Test it out by making a function, `createInitialTags` inside of `seed.js`:

```js
async function createInitialTags() {
  try {
    console.log("Starting to create tags...");

    const [happy, sad, inspo, catman] = await createTags([
      '#happy', 
      '#worst-day-ever', 
      '#youcandoanything',
      '#catmandoeverything'
    ]);

    const [postOne, postTwo, postThree] = await getAllPosts();

    await addTagsToPost(postOne.id, [happy, inspo]);
    await addTagsToPost(postTwo.id, [sad, inspo]);
    await addTagsToPost(postThree.id, [happy, catman, inspo]);

    console.log("Finished creating tags!");
  } catch (error) {
    console.log("Error creating tags!");
    throw error;
  }
}
```

And add it as the last thing to `rebuildDB`:

```js
async function rebuildDB() {
  try {
    client.connect();

    await dropTables();
    await createTables();
    await createInitialUsers();
    await createInitialPosts();
    await createInitialTags(); // new
  } catch (error) {
    console.log("Error during rebuildDB")
    throw error;
  }
}
```

### Update Some Methods

In general we want our posts to have two new keys: `author` and `tags`, to include the information we have been creating.

#### `getPostsByUser`

When we get the posts for a specific user, we will want to include the author and tags for each post.

If modify the original query just to return the `post` id, we can iterate over each post calling our updated `getPostById`, which has all the information we want in it.

Remember, we're avoid a bit of efficiency for readability. This does create **n+1 queries**. which we would have to fix if our app becomes popular, but at this point it seems much more readable to not prematurely optimize.

```js
async function getPostsByUser(userId) {
  try {
    const { rows: postIds } = await client.query(`
      SELECT id 
      FROM posts 
      WHERE "authorId"=${ userId };
    `);

    const posts = await Promise.all(postIds.map(
      post => getPostById( post.id )
    ));

    return posts;
  } catch (error) {
    throw error;
  }
}
```

And now `getPostsByUser` returns something like this:

```js
[
  {
    id: 1,
    title: 'New Title',
    content: 'Updated Content',
    active: true,
    tags: [ [Object], [Object] ],
    author: {
      id: 1,
      username: 'albert',
      name: 'Newname Sogood',
      location: 'Lesterville, KY'
    }
  }
]
```

Also very nice!

#### `getAllPosts`

We can use the same trick we did before to get the posts:

```js
async function getAllPosts() {
  try {
    const { rows: postIds } = await client.query(`
      SELECT id
      FROM posts;
    `);

    const posts = await Promise.all(postIds.map(
      post => getPostById( post.id )
    ));

    return posts;
  } catch (error) {
    throw error;
  }
}
```


Then, your seed output should update:

```bash
Calling getAllPosts
Result: [
  {
    id: 1,
    title: 'First Post',
    content: 'This is my first post. I hope I love writing blogs as much as I love writing them.',
    active: true,
    tags: [ [Object], [Object] ],
    author: {
      id: 1,
      username: 'albert',
      name: 'Newname Sogood',
      location: 'Lesterville, KY'
    }
  },
  {
    id: 2,
    title: 'How does this work?',
    content: 'Seriously, does this even do anything?',
    active: true,
    tags: [ [Object], [Object] ],
    author: {
      id: 2,
      username: 'sandra',
      name: 'Just Sandra',
      location: "Ain't tellin'"
    }
  },
  {
    id: 3,
    title: 'Living the Glam Life',
    content: 'Do you even? I swear that half of you are posing.',
    active: true,
    tags: [ [Object] ],
    author: {
      id: 3,
      username: 'glamgal',
      name: 'Joshua',
      location: 'Upper East Side'
    }
  }
]
```

### Update `seed.js`

First, make sure to export from `index.js` anything we created, and then import them again in `seed.js`.

Then, let's update first our `createInitialTags`:

```js
async function createInitialTags() {
  try {
    console.log("Starting to create tags...");

    const [happy, sad, inspo, catman] = await createTags([
      '#happy', 
      '#worst-day-ever', 
      '#youcandoanything',
      '#catmandoeverything'
    ]);

    const [postOne, postTwo, postThree] = await getAllPosts();

    await addTagsToPost(postOne.id, [happy, inspo]);
    await addTagsToPost(postTwo.id, [sad, inspo]);
    await addTagsToPost(postThree.id, [happy, catman, inspo]);

    console.log("Finished creating tags!");
  } catch (error) {
    console.log("Error creating tags!");
    throw error;
  }
}
```

Now this will populate some post tags.  Realistically we shouldn't need this step, right? We need to update `createPost` to handle creating tags for us.

Let's go do that, then we can update `createInitialPosts` and remove `createInitialTags`:

```js
// in db/index.js
async function createPost({
  authorId,
  title,
  content,
  tags = [] // this is new
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

And then we can update our seeds.

```js
// in db/seed.js
async function createInitialPosts() {
  try {
    const [albert, sandra, glamgal] = await getAllUsers();

    console.log("Starting to create posts...");
    await createPost({
      authorId: albert.id,
      title: "First Post",
      content: "This is my first post. I hope I love writing blogs as much as I love writing them.",
      tags: ["#happy", "#youcandoanything"]
    });

    await createPost({
      authorId: sandra.id,
      title: "How does this work?",
      content: "Seriously, does this even do anything?",
      tags: ["#happy", "#worst-day-ever"]
    });

    await createPost({
      authorId: glamgal.id,
      title: "Living the Glam Life",
      content: "Do you even? I swear that half of you are posing.",
      tags: ["#happy", "#youcandoanything", "#canmandoeverything"]
    });
    console.log("Finished creating posts!");
  } catch (error) {
    console.log("Error creating posts!");
    throw error;
  }
}

async function rebuildDB() {
  try {
    client.connect();

    await dropTables();
    await createTables();
    await createInitialUsers();
    await createInitialPosts();
  } catch (error) {
    console.log("Error during rebuildDB")
    throw error;
  }
}
```

### Updating a Post

Here things are a bit tricky. We need to separately update the post and the tags. The tag list might have some new tags, but it also might be missing some of the tags that used to be part of the post.

```js
const originalPost = {
  title: "title",
  content: "content",
  tags: ["#x", "#y", "#z"]
}

const updatedPost = {
  id: 3,
  title: "new title",
  content: "maybe this changed",
  tags: ["#x", "z", "w"]
}
```

Update needs to take this into account. Again, there are tricky ways of dealing with this, and there are simple ones that use a few more queries. We are going to go the second route.

```js
async function updatePost(id, fields = {}) {
  // read off the tags & remove that field 
  const { tags } = fields; // might be undefined
  delete fields.tags;

  // build the set string
  const setString = Object.keys(fields).map(
    (key, index) => `"${ key }"=$${ index + 1 }`
  ).join(', ');

  // return early if this is called without fields
  if (setString.length === 0) {
    return;
  }

  try {
    const { rows: [ post ] } = await client.query(`
      UPDATE posts
      SET ${ setString }
      WHERE id=${ id }
      RETURNING *;
    `, Object.values(fields));

    // if the user didn't pass in tags to update, return early
    if (tags === undefined) {
      return await getPostById(post.id);
    }

    // make any tags that need to be made
    const tagList = await createTags(tags);
    const tagListIdString = tagList.map(
      tag => `${ tag.id }`
    ).join(', ');

    // now, delete any post_tags from the database which aren't in that tagList, but only those with correct postId
    await client.query(`
      DELETE FROM post_tags
      WHERE tag_id
      NOT IN (${ tagListIdString })
      AND "postId"=$1;
    `, [postId]);
    
    // and create post_tags as necessary
    await addTagsToPost(post.id, tagList);

    return await getPostById(post.id);
  } catch (error) {
    throw error;
  }
}
```

Add to `testDb` a line trying to update a post:

```js
    console.log("Calling updatePost on posts[1], only updating tags");
    const updatePostTagsResult = await updatePost(posts[0].id, {
      tags: ["#youcandoanything", "#redfish", "#bluefish"]
    });
    console.log("Result:", updatePostTagsResult);
```

And you should see this:

```bash
Result: {
  id: 2,
  title: 'How does this work?',
  content: 'Seriously, does this even do anything?',
  active: true,
  tags: [
    { id: 1, name: '#youcandoanything' },
    { id: 9, name: '#redfish' },
    { id: 10, name: '#bluefish' }
  ],
  author: {
    id: 2,
    username: 'sandra',
    name: 'Just Sandra',
    location: "Ain't tellin'"
  }
}
```

The tag `#youcandoanything` is still there, `#sad` is removed, and `#redfish` and `#bluefish` are new.

### `getPostsByTagName`

One last function (is there ever a last function?). Given a tag name let's find all posts with that tag.

Here we will use our trick of mapping post ids to posts from our other functions, but we have to get the correct ids.

One way is to use a double join: connect `posts` to `post_tags`, and then `post_tags` to `tags` by the appropriate keys, then just select the `posts.id` where `tags.name` is correct:

```js
async function getPostsByTagName(tagName) {
  try {
    const { rows: postIds } = await client.query(`
      SELECT posts.id
      FROM posts
      JOIN post_tags ON posts.id=post_tags."postId"
      JOIN tags ON tags.id=post_tags."tagId"
      WHERE tags.name=$1;
    `, [tagName]);
    
    return await Promise.all(postIds.map(
      post => getPostById(post.id)
    ));
  } catch (error) {
    throw error;
  }
} 
```

And test this out inside of your `testDb` function:

```js
    console.log("Calling getPostsByTagName with #happy");
    const postsWithHappy = await getPostsByTagName("#happy");
    console.log("Result:", postsWithHappy);
```

## Wipe your head

At this point you have done _so much work_. 

- We can create and update posts
- We can create tags and attach them to posts
- We can "rehydrate" associated data as part of our functions
- We have handled complex transactions

And most importantly: we haven't lost our cool!

## Up Next

Next week we will be creating a web server. This server will act as an API for application. We will:

- Use `express`, `jwt` and `jwt-express`
- Provide endpoints for `users`
  - POST `/users/register` (sign up)
  - POST `/users/login` (sign in)
  - DELETE `/users/:id` (deactivate account)
- Provide endpoints for `posts`
  - GET `/posts` (see posts)
  - POST `/posts` (create post)
  - PATCH `/posts/:id` (update post)
  - DELETE `/posts/:id` (deactivate post)
- Provide endpoints for `tags`
  - GET `/tags` (list of all tags)
  - GET `/tags/:tagName/posts` (list of all posts with that tagname)

And ultimately deploy to heroku. It'll be a fun time, and you'll have gotten your first API out to the world.