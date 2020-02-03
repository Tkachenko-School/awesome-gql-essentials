# How to set up a GraphQL Server using Node.js, Express & MongoDB


![How to set up a GraphQL Server using Node.js, Express & MongoDB](https://cdn-media-1.freecodecamp.org/images/1*rhpr5EnxrphBwqyTus0jmg.png)

by Leonardo Maldonado

#### The most straightforward way to start with GraphQL & MongoDB.

So you are planning to start with GraphQL and MongoDB. Then you realize how can I set up those two technologies together? Well, this article is made precisely for you. I’ll show you how to set up a GraphQL server using MongoDB. I will show you how you can modularize your GraphQL schema and all this using MLab as our database.

[**All the code from this article is available here.**](https://github.com/leonardomso/graphql-mongodb-server)

So now, let’s get started.

### Why GraphQL?

GraphQL is a query language for your APIs. It was released by Facebook back in 2015 and has gained a very high adoption. It’s the replacement of _REST._

With GraphQL, the client can ask for the exact data that they need and get back exactly what they asked for. GraphQL also uses a JSON-like query syntax to make those requests. All requests go to the same endpoint.

If you’re reading this article, I assume that you know a little bit about GraphQL. If you don’t know, you can learn more about GraphQL [here.](https://graphql.org/)

### Getting started

First, create a folder, then start our project.

    npm init -y 

Then install some dependencies for our project.

    npm install @babel/cli @babel/core @babel/preset-env body-parser concurrently cors express express-graphql graphql graphql-tools merge-graphql-schemas mongoose nodemon

And then _@babel/node_ as a dev dependency:

    npm install --save-dev @babel/node 

#### Babel

Now we’re gonna set up Babel for our project. Create a file called _.babelrc_ in your project folder. Then, put the _@babel/env_ there, like this:

Then go to your package.json and add some scripts:

We’ll have only one script that we’re gonna use in our project.

_“server”_ — It will mainly run our server.

#### Server

Now, in our root folder create the _index.js_ file. It will be where we’re gonna make our server.

First, we’re gonna import all the modules that we’ll use.

Then, we’re gonna create our connect with MongoDB using Mongoose:

What about that _db const_? This is where you’re gonna put your database URL to connect MongoDB. Then you’re gonna say to me: “But, I don’t have a database yet”, yes I got you. For that, we’re using MLab.

MLab is a database-as-a-service for MongoDB, all you need to do is go to their website ([click here](https://mlab.com/)) and register.

After you register, go and create a new database. To use as free, choose this option:

![](https://cdn-media-1.freecodecamp.org/images/1*JDDQnoWbkwJM4R_MFVCB4A.png)

MLab free option to use in our MongoDB Server.


Choose US East (Virginia) as an option, and then give our database a name. After that, our database will show at the home page.

![](https://cdn-media-1.freecodecamp.org/images/1*q25QbcqFOFHIVtxtedxfFw.png)

Our database created with MLab.

Click on our database, then go to _User_ and create a new user. In this example, I’m gonna create a user called _leo_ and password _leoleo1._

After our user is created, on the top of our page, we find two _URLs. O_ne to connect using _Mongo Shell._ The other to connect using a _MongoDB URL._ We copy the second one_._

After that, all you need to do is paste that URL on our _db const_ at _the index.js_ file_._ Our _db const_ would look like this:

#### Express

Now we’re gonna finally start our server. For that, we’ve put some more lines in our _index.js_ and we’re done.

Now, run the command _npm run server_ and go to _localhost:4000/graphql_ and you’ll find a page like this:

![](https://cdn-media-1.freecodecamp.org/images/1*BIhNVFafvKf1kEgDbQ4j9g.png)

The GraphiQL playground.

### MongoDB and Schema

Now, in our root folder, make a folder named _models_ and create a file inside called User.js (yes, with capital U).

Inside of User.js, we’re gonna create our first schema in MongoDB using Mongoose.

Now that we have created our User schema, we’re gonna start with GraphQL.

### GraphQL

In our root folder, we’re gonna create a folder called _graphql._ Inside that folder, we’re gonna create a file called _index.js_ and two more folders: _resolvers_ and _types_.

#### Queries

Queries in GraphQL are the way we ask our server for data. We ask for the data that we need, and it returns exactly that data.

All our queries will be inside our _types_ folder. Inside that folder, create an _index.js_ file and a User folder.

Inside the User folder, we’re gonna create an _index.js_ file and write our queries.

In our types folder, in our _index.js_ file, we’re gonna import all the types that we have. For now, we have the User types.

In case you have more than one type, you import that to your file and include in the _typeDefs_ array.

#### Mutations

Mutations in GraphQL are the way we modify data in the server.

All our mutations will be inside our _resolvers_ folder. Inside that folder, create an _index.js_ file and a User folder.

Inside the User folder, we’re gonna create an _index.js_ file and write our mutations.

Now that all our resolvers and mutations are ready, we’re gonna modularize our schema.

#### Modularizing our schema

Inside our folder called graphql, go to our index.js and make our schema, like this:

Now that our schema is done, go to our root folder and inside our _index.js_ import our schema.

After all that, our schema will end up like this:

### Playing with our queries and mutations

To test our queries and mutations, we’re gonna start our server with the command _npm run server_, and go to _localhost:4000/graphql._

#### Create user

First, we’re gonna create our first user with a mutation:

    mutation {  addUser(id: "1", name: "Dan Abramov", email: "dan@dan.com") {    id    name    email  }}

After that, if the GraphiQL playground returns to you the data object that we created, that means that our server is working fine.

![](https://cdn-media-1.freecodecamp.org/images/1*U2vN26hF2kLC_8JXtUMvyA.png)

The data object that GraphiQL Playground returned for us.

To make sure, go to MLab, and inside of our _users_ collection, check if there is our data that we just created.

![](https://cdn-media-1.freecodecamp.org/images/1*kY0ftzycpo0QwjRgc0eFUA.png)

In our first mutation, we created a user.

After that, create another user called “Max Stoiber”. We add this user to make sure that our mutation is working fine and we have more than one user in the database.

#### Delete user

To delete a user, our mutation will go like this:

    mutation {  deleteUser(id: "1", name: "Dan Abramov", email: "dan@dan.com") {    id    name    email  }}

Like the other one, if the GraphiQL playground returns to you the data object that we created, that means that our server is working fine.

#### Get all users

To get all users, we’re gonna run our first query like this:

    query {  users {    id    name    email  }}

Since we only have one user, it should return that exact user.

![](https://cdn-media-1.freecodecamp.org/images/1*r_o7jCMP0fQdiFDsozGoWg.png)

All our users will be returned.

#### Get a specific user

To get a specific user, this will be the query:

    query {  user(id: "2"){    id    name    email  }}

That should return the exact user.

![](https://cdn-media-1.freecodecamp.org/images/1*A6UZ-KVEhOthrgFO4qZxcg.png)

The exact user that we asked.

### And we’re done!

Our server is running, our queries and mutations are working fine, we’re good to go and start our client. You can start with _create-react-app._ In your root folder just give the command _create-react-app client_ and then, if you run the command **_npm run dev_**, our _server_ and _client_ will run concurrently!