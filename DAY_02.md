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

- Add `deleteUser` method which
  - takes a user id
  - expects the `active` field to be updated to `false` (soft delete)

- Update `seed.js` to test deactivating a user

- Create a new table, `posts`
  - `id` of type `SERIAL`, which is a `PRIMARY KEY`
  - `title`, `splashUrl` both `VARCHAR`
  - `content` of type `TEXT`
  - `authorId` of type `INTEGER`, and `REFERENCES users(id)`
  - `active` boolean

- Add the following methods:
  - `createPost({ title, content, splashUrl, authorId })`
  - `deletePost(id)` - soft delete
  - `updatePost({ id, title content, splashUrl, authorId })`
  - `getAllPosts()` - only active posts
  - `getPostsByUser(id)` - only active posts

- Add some seeds and tests to `seed.js`
