# JuiceBox

We are going to make a light [tumblr](https://www.tumblr.com/) clone. Tumblr is a website that lets users write blog posts, and tag the posts.

In order to do this we will write our program over three total phases:

## The Database

Our database will be built on [postgresql](https://www.postgresql.org/), an implementation of SQL. SQL itself is a particular implementation of a [relational database](https://en.wikipedia.org/wiki/Relational_database).

In short the data we use is *relational*, or has natural relationships between different data types:

- Our users create posts
- Our posts have many tags
- Our tags might belong to many posts

Here is an [ERD](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model) showing the relationships, and which fields they'll have:

![juicebox-erd](https://drive.google.com/open?uc=1gKK19DdzsmBcoG4J8UPfa3j_Yfmeg_sx)

## The Web Server

Once we've written our database adapter and relevant functions for the CRUD actions we envision, we will build our web server. We will use Express, and endeavor to make a clean API for consumption.

We will also use [JSON Web Tokens (JWT)](https://scotch.io/tutorials/the-anatomy-of-a-json-web-token) for user authentication, much like we did in the Stranger's Things project.

Our goal is to build out a good API for our front-end to consume, as well as serve up the front end from our express server's public folder.

## The Front-End

Here our goal is to think in components, and use a slightly different pattern when building out our application.