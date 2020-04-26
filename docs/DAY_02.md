# Day 02

Goals for day 2:

- Flesh out the `users` Table
  - add in `location`, and `name`, both `VARCHAR`
  - add in `active`, a boolean

- Update `createUser` to reflect these changes. Set `active` to `true` on create.

- Add `updateUser` method which:
  - takes an object (which things need to be updated, plus an `id` key )

      ```js
      updateUser({
        id: 5,
        username: "new_user_name",
        location: "brooklyn, ny"
      });
      ```

  - And expects that the database will update accordingly

      ```sql
      UPDATE users
      SET username='new_user_name', location='brooklyn, ny'
      WHERE id=5;
      ```

- Update `seed.js` to add required fields, and in `testDB` make a call to `updateUser` to ensure it works

- Create a new table, `posts`
  - `id` of type `SERIAL`, which is a `PRIMARY KEY`
  - `content` of type `TEXT`
  - `"authorId"` of type `INTEGER`, and `REFERENCES users(id)`
  - `active` boolean

- Add the following methods:
  - `createPost({ title, content, authorId })`
  - `updatePost(id, {  title, content, active })`
  - `getAllPosts()` - only active posts
  - `getPostsByUser(userId)` - only active posts

- Add some seeds and tests to `seed.js`

## Starting off

Make sure to run `npm run seed:dev` from your local repo at the start of development.

## Updating our users table

We need to add the following fields:

- `name`, `VARCHAR(255)`
- `location`, `VARCHAR(255)`
- `active`, `BOOLEAN DEFAULT true`

So go back to your table definition in `db/seed.js` and update the table definition accordingly.

Once you save, you should get an error!

```bash
Error creating users!
error: null value in column "name" violates not-null constraint
```

Neat, our database is protecting us! Go ahead and update the three users in our `createInitialUsers` to have `name` and `location` properties.

But wait... our `createUser` function doesn't do anything with those properties! We'll have to update that too, which means updating the SQL query. 

At this point I like having my destructuring go vertical:

```js
async function createUser({ 
  username, 
  password,
  name,
  location
}) {
  try {
    const { rows } = await client.query(`
      INSERT INTO users(username, password, name, location) 
      VALUES($1, $2, $3, $4) 
      ON CONFLICT (username) DO NOTHING 
      RETURNING *;
    `, [username, password, name, location]);

    return rows;
  } catch (error) {
    throw error;
  }
}
```

The good news is that we don't have to pass in a value for `active`, our table will set it for us when the user is inserted because of that default value.

And if you look at your logs, you'll see that we're still only getting `id` and `username` for our users:

```bash
getAllUsers: [
  { id: 1, username: 'albert' },
  { id: 2, username: 'sandra' },
  { id: 3, username: 'glamgal' }
]
```

So we also need to update `getAllUsers`: add `name`, `location`, and `active` to the fields we want to get when we call `getAllUsers`. 

Your script should output something like this when you're done:

```bash
getAllUsers: [
  {
    id: 1,
    username: 'albert',
    name: 'Al Bert',
    location: 'Sidney, Australia',
    active: true
  },
  {
    id: 2,
    username: 'sandra',
    name: 'Just Sandra',
    location: "Ain't tellin'",
    active: true
  },
  {
    id: 3,
    username: 'glamgal',
    name: 'Joshua',
    location: 'Upper East Side',
    active: true
  }
]
```

### `updateUser`

Now we want to write a method that will allow us to change a user should they want to update their profile.

**Quick Pep Talk**: Updating is *super duper* hard. There's a lot that can go wrong, and we have to carefully build the query. To that end, let me help you out here:

```js
async function updateUser(id, fields = {}) {
  // build the set string
  const setString = Object.keys(fields).map(
    (key, index) => `"${ key }"=$${ index + 1 }`
  ).join(', ');

  // return early if this is called without fields
  if (setString.length === 0) {
    return;
  }

  try {
    const result = await client.query(`
      UPDATE users
      SET ${ setString }
      WHERE id=${ id }
      RETURNING *;
    `, Object.values(fields));

    return result;
  } catch (error) {
    throw error;
  }
}
```

The first thing we do is create the things that will go next to `SET` in our query. A good update query would look like this:

```sql
UPDATE users
SET "name"='new name', "location"='new location'
WHERE id=2;
```

Each key in the `fields` object should match a column name for our table, and each value should be the new value for it. We use map to turn each key into a string that looks like `"keyName"=$3` where the key name is in quotes (in case the table colum is case sensitive), and we have a parameter whose numeric value is one greater than the index of that particular key.

Once we build the set string, as long as the `fields` object had something in it, we call our query.  We can safely interpolate the `id` since we will be passing it in when we call `updateUser`.

#### Make sure to export our function

Add `updateUser` to `module.exports`

#### Test `updateUser`

Now over in `db/seeds.js` we should test it out! Make sure to destructure it from our `require` statement at the top, then go modify `testDB`:

```js
async function testDB() {
  try {
    console.log("Starting to test database...");

    console.log("Calling getAllUsers")
    const users = await getAllUsers();
    console.log("Result:", users);

    console.log("Calling updateUser on users[0]")
    const updateUserResult = await updateUser(users[0].id, {
      name: "Newname Sogood",
      location: "Lesterville, KY"
    });
    console.log("Result:", updateUserResult);

    console.log("Finished database tests!");
  } catch (error) {
    console.error("Error testing database!");
    throw error;
  }
}
```

And you should see this:

```bash
Calling updateUser on users[0]
Result: Result {
  command: 'UPDATE',
  rowCount: 1,
  oid: null,
  rows: [
    {
      id: 1,
      username: 'albert',
      password: 'bertie99',
      name: 'Newname Sogood',
      location: 'Lesterville, KY',
      active: true
    }
  ],
# ... and so much more
```

Go destructure the returned user `rows` from the result back in `updateUser` and return it.

```js
// we can use advanced destructuring here
const { rows: [ user ] } = await client.query(`

`, []);

// ...

return user;
```

And your output should look like this:

```bash
Calling updateUser on users[0]
Result: {
  id: 1,
  username: 'albert',
  password: 'bertie99',
  name: 'Newname Sogood',
  location: 'Lesterville, KY',
  active: true
}
```

Just the updated user, with new name & location!

#### Update `createUser`

Go back and destructure the newly created user from our query in our `createUser` function the same way we just did. Make sure to return it rather than the rows array.

## Posts

Alright! Now that we've sorted out our users, we can start sorting out our posts.

### First, create the table

Our posts table needs to have the following columns, and please note the quotes around `authorId`... to keep the case the same, postgres will require you to do that:

- `id`, `SERIAL PRIMARY KEY`
- `"authorId"`, `INTEGER REFERENCES users(id) NOT NULL`
- `title`, `VARCHAR(255) NOT NULL`
- `content`, `TEXT NOT NULL`
- `active`, `BOOLEAN DEFAULT true`

You can write the `CREATE TABLE posts();` command inside `db/seed.js`.  Once you successfully create the table, if you run `db/seed.js` twice, it will throw an interesting new error:

```bash
Error dropping tables!
error: cannot drop table users because other objects depend on it
```

This is because of the `REFERENCES` on `users(id)`. It's not nice for a table to go away when something else relies on it. Add `DROP TABLE IF EXISTS posts;` **before** dropping the `users` table inside of `dropTables` to fix this.

### Then, create the methods

Inside of `db/index.js` write and export these functions:

#### `createPost`

This should mimic `createUser` with very few changes.

```js
async function createPost({
  authorId,
  title,
  content
}) {
  try {

  } catch (error) {
    throw error;
  }
}
```

#### `updatePost`

This should mimic `updateUser` with very few changes. We aren't making it possible to update the foreign key of `authorId`, since... well, authorship of a post shouldn't change hands.

```js
async function updatePost(id, {
  title,
  content,
  active
}) {
  try {

  } catch (error) {
    throw error;
  }
}
```

#### `getAllPosts`

This should mimic `getAllUsers` with very few changes.

```js
async function getAllPosts() {
  try {

  } catch (error) {
    throw error;
  }
}
```

#### `getPostsByUser`

This is the one that's a bit harder, because we need to add a where clause to our search, but otherwise it's not that bad:

```js
async function getPostsByUser(userId) {
  try {
    const { rows } = client.query(`
      SELECT * FROM posts
      WHERE "authorId"=${ userId };
    `);

    return rows;
  } catch (error) {
    throw error;
  }
}
```

#### For the users: `getUserById`

We can use `getPostsByUser` to make two queries inside the same method. We want a way to get **both** a user and their posts, like so:

```js
{
  id: 12,
  username: 'sal',
  name: 'salvatore',
  location: 'brooklyn, ny',
  posts: [
    // post objects here...
  ]
}
```

So we can write the following method:

```js
async function getUserById(userId) {
  // first get the user
  // if it doesn't exist, return null

  // if it does:
  // delete the 'password' key from the returned object
  // get their posts (use getPostsByUser)
  // then add the posts to the user object with key 'posts'
  // return the user object
}
```

### Then, test the methods

Start by creating a new function `createInitialPosts`, and call it inside of `rebuildDB` just after `createInitialUsers`:

```js
async function createInitialPosts() {
  try {
    const [albert, sandra, glamgal] = await getAllUsers();
    
    await createPost({
      authorId: albert.id,
      title: "First Post",
      content: "This is my first post. I hope I love writing blogs as much as I love writing them."
    });

    // a couple more
  } catch (error) {
    throw error;
  }
}
```

Then work on `testDb`. You can use my final code, or write your own tests.

```js
async function testDB() {
  try {
    console.log("Starting to test database...");

    console.log("Calling getAllUsers");
    const users = await getAllUsers();
    console.log("Result:", users);

    console.log("Calling updateUser on users[0]");
    const updateUserResult = await updateUser(users[0].id, {
      name: "Newname Sogood",
      location: "Lesterville, KY"
    });
    console.log("Result:", updateUserResult);

    console.log("Calling getAllPosts");
    const posts = await getAllPosts();
    console.log("Result:", posts);

    console.log("Calling updatePost on posts[0]");
    const updatePostResult = await updatePost(posts[0].id, {
      title: "New Title",
      content: "Updated Content"
    });
    console.log("Result:", updatePostResult);

    console.log("Calling getUserById with 1");
    const albert = await getUserById(1);
    console.log("Result:", albert);

    console.log("Finished database tests!");
  } catch (error) {
    console.log("Error during testDB");
    throw error;
  }
}
```