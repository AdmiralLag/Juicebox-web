# Day 03

Goals for day 3:

- Create a new table, `tags`
  - `id` of type `SERIAL`, which is a `PRIMARY KEY`
  - `name` of type `VARCHAR`

- Similarly create a new table, `post_tags`
  - `id` of type `SERIAL` which is a `PRIMARY KEY`
  - `postId`
  - `tagId`
  - add a `UNIQUE` constraint on `("postId", "tagId")` to prevent the same pair from being created

- Add the following methods to `db/index.js`:
  - `getTags()` - all tags
  - `getTagsForPost(postId)` - get tags for a single post
  - `createTag(name)` (creates an entry, if possible)
  - `createPostTags(postId, [ array of tagId ])`

- Add some seeds and tests to `seed.js`
  
    ```js
    const max = await createUser();
    const post = await createPost(); // for max
    const tags = await Promise.all(createTag(), createTag(), ...);
    const postTags = createPostTags(post.id, tags);
    ```

- Then update `getPostById` and `getPosts` to include the tags in the posts

    - Basically for getPosts - do a full lookup of posts *and* tags *and* post_tags and add the correct tags to the posts