# Day 6

Let's remember the original estimation of routes we are trying to build:

- Provide endpoints for `users`
  - **DONE**: POST `/users/register` (sign up)
  - **DONE**: POST `/users/login` (sign in)
  - DELETE `/users/:id` (deactivate account)
- Provide endpoints for `posts`
  - **DONE-ish**: GET `/posts` (see posts)
  - POST `/posts` (create post)
  - PATCH `/posts/:id` (update post)
  - DELETE `/posts/:id` (deactivate post)
- Provide endpoints for `tags`
  - **DONE**: GET `/tags` (list of all tags)
  - GET `/tags/:tagName/posts` (list of all posts with that tagname)

We will leave user deactivation and post deactivation to the end, since they will trigger a number of updates we need to do to our database, but for now let's get started with our `posts`.

## Create Posts
