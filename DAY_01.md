Day 01

## Postgres Setup - Ubuntu/Linux

This is crucial for anyone using an Ubuntu/Linux install rather than a MacOS install.

Postgres uses a file, `pg_hba.conf`, to control who can access the databases, and what they can do when they do it. We will have to edit that file to 

```bash
# locate the file using the find command
find / -name pg_hba.conf
# you might get some warnings about directories that find can't access, ignore them

# eventually it will find something like this:
# /etc/postgresql/9.6/main/pg_hba.conf

# use that file path from above here:
sudo vi /etc/postgresql/9.6/main/pg_hba.conf 
# we need to edit this file as root
```

Once your in the file editor, use your arrow keys to navigate to lines that look *something* like this:

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                peer
local   all             all                                     md5
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5
```

For now, change the `METHOD` for the ones with `TYPE` equal to `local` or `host` to `trust`.  That will mean that your local development machine won't need login/password combinations from your node programs to access the database.

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                trust
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 strust
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5
```

To change the file in `vi`:

- Type the letter `i`, which puts you in insert mode
- Now you can navigate, use backspace, and type

Once you make changes to the relevant lines above:

- Hit your escape key to leave insert mode
- Type a colon (`:`)
- Type `wq` and hit return

This will save the file, and return you to the terminal. Once there, type this:

```bash
sudo service postgresql restart
```

And you should be good to go.  One way to check is this:

```bash
# go into your postgres db as the user named postgres
psql -U postgres 

# CORRECT SETTINGS:
# psql (9.6.17)
# Type "help" for help.
#
# postgres=# 

# INCORRECT SETTINGS:
# psql: FATAL:  Peer authentication failed for user "postgres"

```

## Initial Structure & Remote Setup

Initialize file structure:

```
~/curriculum/juicebox
├── public
|   ├── index.html
|   ├── app.js
|   └── style.css
├── db
|   ├── index.js
|   └── seed.js
├── index.js
└── .gitignore
```

Set up initial NPM Project and Git repository:

```bash
cd ~/curriculum/juicebox

npm init -y

git init

git add .

git commit -m "first commit"
```

Then go to GitHub and create a remote repository called juicebox. Come back and attach the remote to your local repo, then push the local changes to the remote repo.

```bash
git remote add origin https://github.com/YOUR_USER_NAME/juicebox.git
git push -u origin master
```

## Playing with Databases

Then, we need to get our database setup:

```bash
createdb juicebox-dev

psql juicebox-dev

juicebox-dev=# \dt
No relations found.
```

Then, for now, let's play with SQL directly from this prompt:

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username varchar(255) UNIQUE NOT NULL,
  password varchar(255) NOT NULL
);
```

You can peek at the table you just defined like this:

```bash
juicebox-dev=# \d users
                                  Table "public.users"
  Column  |          Type          |                     Modifiers                      
----------+------------------------+----------------------------------------------------
 id       | integer                | not null default nextval('users_id_seq'::regclass)
 username | character varying(255) | not null
 password | character varying(255) | not null
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_username_key" UNIQUE CONSTRAINT, btree (username)
```

And let's create a few `users`:

```sql
INSERT INTO users (username, password)
VALUES
  ('albert', 'bertie99'),
  ('sandra', '2sandy4me'),
  ('glamgal', 'soglam');

# INSERT 0 3
```

You can now try to pull back up a user:

```sql
SELECT * FROM users; 

 id | username | password  
----+----------+-----------
  1 | albert   | bertie99
  2 | sandra   | 2sandy4me
  3 | glamgal  | soglam
(3 rows)

SELECT id, username FROM users WHERE username='albert' AND password='bertie99';

 id | username 
----+----------
  1 | albert
(1 row)

SELECT id, username FROM users WHERE username='albert' AND password='bertiw99';

 id | username 
----+----------
(0 rows)

\q
```

Great, we have a way to **C**reate users, and to **R**ead a user. That's about half-way to full CRUD.

The big problem, however, is that we are doing this manually. The reason it's a problem is we want to be able to do this, on command, when a user interacts with our website.

## Playing With Databases in Node

Now that we know a few commands that work with PostgreSQL, we can transfer them to our node application.

```bash
npm install pg # node's postgresql adapter

npm install nodemon --save-dev # live reload
```

And edit the `"scripts"` in our `package.json`:

```json
{
  "scripts": {
    "seed:dev": "nodemon ./db/seed.js"
  }
}
```

We are going to put our `seed.js` file on live reload so that we can see what's happening as we make changes. For now we will rely on logging to see what's happening under the hood as we go.

```js
// inside db/index.js
const { Client } = require('pg'); // imports the pg module

// supply the db name and location of the database
const client = new Client('postgres://localhost:5432/juicebox-dev');

module.exports = {
  client,
}
```

For now this will get us access to the database, so let's go edit the `seed.js` file:

```js
// inside db/seed.js

// grab our client with destructuring from the export in index.js
const { client } = require('./index');

async function testDB() {
  try {
    // connect the client to the database, finally
    client.connect();

    // queries are promises, so we can await them
    const result = await client.query(`SELECT * FROM users;`);

    // for now, logging is a fine way to see what's up
    console.log(result);
  } catch (error) {
    console.error(error);
  } finally {
    // it's important to close out the client connection
    client.end();
  }
}

testDB();
```

Then run our script:

```bash
npm run seed:dev

Result {
  command: 'SELECT',
  rowCount: 3,
  oid: null,
  rows: [
    { id: 1, username: 'albert', password: 'bertie99' },
    { id: 2, username: 'sandra', password: '2sandy4me' },
    { id: 3, username: 'glamgal', password: 'soglam' }
  ],
  fields: [
    Field {
      name: 'id',
      tableID: 17210,
      columnID: 1,
      dataTypeID: 23,
      dataTypeSize: 4,
      dataTypeModifier: -1,
      format: 'text'
    },
    Field {
      name: 'username',
      tableID: 17210,
      columnID: 2,
      dataTypeID: 1043,
      dataTypeSize: -1,
      dataTypeModifier: 259,
      format: 'text'
    },
    Field {
      name: 'password',
      tableID: 17210,
      columnID: 3,
      dataTypeID: 1043,
      dataTypeSize: -1,
      dataTypeModifier: 259,
      format: 'text'
    }
  ],
  _parsers: ...
  ...
}

[nodemon] clean exit - waiting for changes before restart
```

Nice! Our `SELECT` executed, found three rows (the three we created earlier on), and we were able to access it. It also shows us the structure of a query result.

In general we will be accessing the `rows` field. Make this change:

```js
async function testDB() {
  try {
    client.connect();

    const { rows } = await client.query(`SELECT * FROM users;`);
    console.log(rows);
  } catch (error) {
    console.error(error);
  } finally {
    client.end();
  }
}
```

Save and look in the terminal:

```bash
[nodemon] restarting due to changes...
[nodemon] starting `node ./db/seed.js`
[
  { id: 1, username: 'albert', password: 'bertie99' },
  { id: 2, username: 'sandra', password: '2sandy4me' },
  { id: 3, username: 'glamgal', password: 'soglam' }
]
[nodemon] clean exit - waiting for changes before restart
```

Nice!

## All the Helper Functions

Back in our `index.js` file, let's start building some helper functions that we might use throughout our application:

```js
// inside db/index.js

async function getAllUsers() {
  const { rows } = await client.query(`SELECT id, username FROM users;`);

  return rows;
}

// and export them
module.exports = {
  client,
  getAllUsers,
}
```

And back in our `seed.js`:

```js
// inside db/seed.js

const {
  client,
  getAllUsers
} = require('./index');

async function testDB() {
  try {
    client.connect();

    const users = await getAllUsers();
    console.log(users);
  } catch (error) {
    console.error(error);
  } finally {
    client.end();
  }
}
```

## The Pattern

In general, in our `db/index.js` file we should provide the utility functions that the rest of our application will use. We will call them from the `seed` file, but also from our main application file. 

That is where we are going to listen to the front-end code making AJAX requests to certain routes, and will need to make our own requests to our database.

![Image of Pattern]()

## Seeding

When we talk about seeding, we mean one or more of the following:

- Making sure that the tables have correct definitions
- Making sure that the tables have no unwanted data
- Making sure that the tables have some data for us to play with
- Making sure that the tables have necessary user-facing data

We are going to primarily use our seed file to build/rebuild the tables, and to fill them with some starting data. We need a programmatic way to do this, rather than having to connect directly to the SQL server and directly type in the queries by hand.

Let's work a bit on these goals, starting with dropping and rebuilding the tables:

```js
// inside db/seed.js

// this function should call a query which drops all tables from our database
async function dropTables() {
  try {
    await client.query(`
    
    `);
  } catch (error) {
    throw error; // we pass the error up to the function that calls dropTables
  }
}

// this function should call a query which creates all tables for our database 
async function createTables() {
  try {
    await client.query(`
    
    `);
  } catch (error) {
    throw error; // we pass the error up to the function that calls createTables
  }
}

async function rebuildDB() {
  try {
    client.connect();

    await dropTables();
    await createTables();
  } catch (error) {
    console.error(error);
  } finally {
    client.end();
  }
}

rebuildDB();
```

### Start with `dropTables`

In SQL, there is a defined way to drop a table:

```sql
DROP TABLE table_name;
```

Connect to your server, and try it for our `users` table, and then try it a second time.

```sql
DROP TABLE users;
DROP TABLE

DROP TABLE users;
ERROR:  table "users" does not exist
```

In general, we want our commands to execute error free whenever possible. SQL gives us a way to add a condition to our `DROP TABLE` request... add `IF EXISTS` to it:

```sql
DROP TABLE IF EXISTS users;
NOTICE:  table "users" does not exist, skipping
DROP TABLE
```

Now instead of throwing an error, we see that we get a `NOTICE` instead! It tries it, it fails, then it moves forward with whatever the rest of our query is. In this case there is no "rest" of the query.

We can move this into `dropTables`:


```js
// in db/seed.js

async function dropTables() {
  try {
    await client.query(`
      DROP TABLE IF EXISTS users;
    `);
  } catch (error) {
    throw error;
  }
}
```

### Now finish `createTables`

We've already seen how to create our users table, so we can move that into our seed file:

```js
// in db/seed.js

async function createTables() {
  try {
    await client.query(`
      CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        username varchar(255) UNIQUE NOT NULL,
        password varchar(255) NOT NULL
      );
    `);
  } catch (error) {
    throw error;
  }
}
```

### And clean up `seed.js` in general

Let's remove the `finally` from `rebuildDB` and `testDB`, and add some useful logs to each of our functions inside of `seed.js`:

```js
// db/seed.js

const {
  client,
  getAllUsers
} = require('./index');

async function dropTables() {
  try {
    console.log("Starting to drop tables...");

    await client.query(`
      DROP TABLE IF EXISTS users;
    `);

    console.log("Finished dropping tables!");
  } catch (error) {
    console.error("Error dropping tables!");
    throw error;
  }
}

async function createTables() {
  try {
    console.log("Starting to build tables...");

    await client.query(`
      CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        username varchar(255) UNIQUE NOT NULL,
        password varchar(255) NOT NULL
      );
    `);

    console.log("Finished building tables!");
  } catch (error) {
    console.error("Error building tables!");
    throw error;
  }
}

async function rebuildDB() {
  try {
    client.connect();

    await dropTables();
    await createTables();
  } catch (error) {
    throw error;
  }
}

async function testDB() {
  try {
    console.log("Starting to test database...");

    const users = await getAllUsers();
    console.log("getAllUsers:", users);

    console.log("Finished database tests!");
  } catch (error) {
    console.error("Error testing database!");
    throw error;
  }
}


rebuildDB()
  .then(testDB)
  .catch(console.error)
  .finally(() => client.end());
```

Save and you should see output like this:

```bash
[nodemon] restarting due to changes...
[nodemon] starting `node ./db/seed.js`
Starting to drop tables...
Finished dropping tables!
Starting to build tables...
Finished building tables!
Starting to test database...
getAllUsers: []
Finished database tests!
[nodemon] clean exit - waiting for changes before restart
```

## So Empty

Well, now every time our seed file runs, we have an empty database. Let's build a function in `db/index.js` that we can use in `db/seed.js`:

```js
// in db/index.js

async function createUser({ username, password }) {
  try {
    const result = await client.query(`
    
    `);

    return result
  } catch (error) {
    throw error;
  }
}
```

We've previously written a SQL query that creates users, so we should be able to drop that in here. One thing that the `pg` package does for us is gives us a nice way to interpolate values:

```js
client.query(`
  INSERT INTO users(username, password) VALUES ($1, $2);
`, [ "some_name", "some_password" ]);
```

So let's use that for creating users:

```js
// in db/index.js

async function createUser({ username, password }) {
  try {
    const result = await client.query(`
      INSERT INTO users(username, password)
      VALUES ($1, $2);
    `, [username, password]);

    return result;
  } catch (error) {
    throw error;
  }
}

// later
module.exports = {
  // add createUser here!
}
```

### Test it out

We can now bring that exported function into `seed.js`, and make it part of our `rebuildDB`:

```js
// db/seed.js

const { 
  // other imports,
  createUser
} = require('./index');

// new function, should attempt to create a few users
async function createInitialUsers() {
  try {
    console.log("Starting to create users...");

    const albert = await createUser({ username: 'albert', password: 'bertie99' });

    console.log(albert);

    console.log("Finished creating users!");
  } catch(error) {
    console.error("Error creating users!");
    throw error;
  }
}

// then modify rebuildDB to call our new function
async function rebuildDB() {
  try {
    client.connect();

    await dropTables();
    await createTables();
    await createInitialUsers();
  } catch (error) {
    throw error;
  }
}

// old stuff below here! 
```

Once you save, you should see something like this:

```bash
# more up here
Result {
  command: 'INSERT',
  rowCount: 1,
  oid: 0,
  rows: [],
  fields: [],
  _parsers: undefined,
  _types: TypeOverrides {
    _types: {
      getTypeParser: [Function: getTypeParser],
      setTypeParser: [Function: setTypeParser],
      arrayParser: [Object],
      builtins: [Object]
    },
    text: {},
    binary: {}
  },
  RowCtor: null,
  rowAsArray: false
}
Finished creating users!
Starting to test database...
getAllUsers: [ { id: 1, username: 'albert' } ]
Finished database tests!
```

Taking in the output, what does it all mean?

1. We clearly inserted "albert" into the database, we see the result of doing so in `getAllUsers`
2. However, if we tried to grab anything from the `rows` of the result, there would be nothing for us to use.

Further, try this:

```js
// add to createInitialUsers
const albertTwo = await createUser({ username: 'albert', password: 'imposter_albert' });
```

And watch what happens, you'll get an output like this:

```bash
Error creating users!
error: duplicate key value violates unique constraint "users_username_key"
    at Connection.parseE (/home/matthewfshort/juicebox/node_modules/pg/lib/connection.js:581:48)
    at Connection.parseMessage (/home/matthewfshort/juicebox/node_modules/pg/lib/connection.js:380:19)
    at Socket.<anonymous> (/home/matthewfshort/juicebox/node_modules/pg/lib/connection.js:116:22)
    at Socket.emit (events.js:315:20)
    at addChunk (_stream_readable.js:297:12)
    at readableAddChunk (_stream_readable.js:273:9)
    at Socket.Readable.push (_stream_readable.js:214:10)
    at TCP.onStreamRead (internal/stream_base_commons.js:186:23) {
  name: 'error',
  length: 207,
  severity: 'ERROR',
  code: '23505',
  detail: 'Key (username)=(albert) already exists.',
  hint: undefined,
  position: undefined,
  internalPosition: undefined,
  internalQuery: undefined,
  where: undefined,
  schema: 'public',
  table: 'users',
  column: undefined,
  dataType: undefined,
  constraint: 'users_username_key',
  file: 'nbtinsert.c',
  line: '433',
  routine: '_bt_check_unique'
}
```

We're violating the unique key constraint. Thankfully we can change the SQL query in `createUser` to fix both problems.

```js
async function createUser({ username, password }) {
  try {
    const result = await client.query(`
      INSERT INTO users(username, password) 
      VALUES($1, $2) 
      ON CONFLICT (username) DO NOTHING 
      RETURNING *;
    `, [username, password]);

    return result;
  } catch (error) {
    throw error;
  }
}
```

#### First Albert

After we insert `albert` you'll see this if you log the result:

```bash
Result {
  command: 'INSERT',
  rowCount: 1,
  oid: 0,
  rows: [ { id: 1, username: 'albert', password: 'bertie99' } ],
  fields: [
    Field {
      name: 'row',
      tableID: 0,
      columnID: 0,
      dataTypeID: 2249,
      dataTypeSize: -1,
      dataTypeModifier: -1,
      format: 'text'
    }
  ],
  _parsers: [ [Function: noParse] ],
  _types: TypeOverrides {
    _types: {
      getTypeParser: [Function: getTypeParser],
      setTypeParser: [Function: setTypeParser],
      arrayParser: [Object],
      builtins: [Object]
    },
    text: {},
    binary: {}
  },
  RowCtor: null,
  rowAsArray: false
}
```

Hooray! We see the result of inserting gives us back a row object. We can use this to create a return to future front ends.

#### Second Albert

Now if you try to try to insert `albertTwo`, you'll get the following (logging the result):

```bash
Result {
  command: 'INSERT',
  rowCount: 0,
  oid: 0,
  rows: [],
  fields: [
    Field {
      name: 'row',
      tableID: 0,
      columnID: 0,
      dataTypeID: 2249,
      dataTypeSize: -1,
      dataTypeModifier: -1,
      format: 'text'
    }
  ],
  _parsers: [ [Function: noParse] ],
  _types: TypeOverrides {
    _types: {
      getTypeParser: [Function: getTypeParser],
      setTypeParser: [Function: setTypeParser],
      arrayParser: [Object],
      builtins: [Object]
    },
    text: {},
    binary: {}
  },
  RowCtor: null,
  rowAsArray: false
}
```

The lack of a row object on insert is great, it can now tell us that nothing was inserted.

### Clean up `createUser`

Just go back and clean up `createUser` to return the rows, not the full result. It should mimic `getAllUsers`, so feel free to look there for inspiration.

### Finish `createInitialUsers`

Remove `albertTwo`, and put back in `sandra` and `glamgal` from our initial seeds.

## Wrapping Up

Good job today.  It was hard work, but you've now:

- Created a database in PSQL
- Created a table for that database using PSQL command line
- Created entries for that database using PSQL command line

And also:

- Used `node-postgres` to start setting up the basis for an excellent back-end API for our current project.

### Git Commit

We never touched our `.gitignore`: make sure to go into it and add `node_modules`. 

Then `git add .`, `git commit -m "my commit message"`, and `git push origin master`
