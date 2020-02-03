# Building a GraphQL Server with Node.js and Express

![](https://cdn-images-1.medium.com/max/1600/1*a4sGHbJDDttlJI3xWg-28w.jpeg)

Follow me on [Twitter](https://twitter.com/chris_noring), happy to take your suggestions on topics or improvements /Chris

This article is part of a series on GraphQL:

*   Building a GraphQL server using Node.js and Express, **we are here**
*   [Building a GraphQL server using Node.js and the Apollo Framework](https://dev.to/azure/creating-a-graphql-server-withapollo-jjj)
*   [Consuming an Apollo GraphQL server using React](https://dev.to/softchris/consuming-an-apollo-graphql-server-usingreact-3kk4)

> GraphQL enables us to build a server that a client can query in any way they want. I’m of course talking about content negotiation, the ability to ask the server exactly for the fields or resources they want, at any depth.

In a previous article [Building your first GraphQL Server](https://medium.com/@noringc/building-your-first-graphql-server-d5c4f88f5e82), I’ve covered what different artifacts and types that make out a GraphQL server. In this article, we will focus more on how to build a service, powered by GraphQL that clients can consume. We will also introduce a playground environment called GraphiQL that gives us a UI in which we can test our queries out.

In this article we will cover:

*   **why Graphql**, Graphql is a nice new technology let's try to cover why it's relevant and why it will make building APIs fun and easy
*   **building blocks**, let's go through the building blocks that you need to build the server side of Graphql
*   **building a server**, we will use Node.js, Express, amd a library called express-graphql to make it possible
*   **querying**, we will cover different ways of querying our server like normal queries, paramterized queries and also how to change data with mutations

## Why GraphQL

There are many reasons for choosing GraphQL over REST:

*   **the data you need**, As with all techniques using content negotiation you get the ability to query for just the data you need that means you get exactly the columns you ask for and that keeps the returning answer to a minimum. Especially in todays world with mobile first and 3G/4G connections, keeping the data responses to a minimum in size is a really great thing.
*   **one endpoint**, as soon as you want data specific data from an endpoint you query that specific endpoint. What if the data you need is something you need to puzzle together from more than one endpoint? At that point, you carry out a bunch of calls or build a new endpoint. Regardless of approach you choose, you need to spend time managing and knowing your endpoints. GraphQL shines here as it is just one endpoint.
*   **serialization**, when you call a REST you get the response JSON you get. However, you might need to do some extra _massage_ to your data like renaming columns, for example, to better suit your application. With GraphQL you can specify this in the query itself
*   **go at depth**, normally with REST it's easy a specific thing like an Order. What if you wanted to fetch the Order Items on that Order or even the Products the Customer bough? Most likely you would have to make several calls or make a specific reporting query for this to avoid extra roundtrips. With GraphQL you can query as deep as you need in the Graph and bring out the data you need at any depth. Of course, doing this in an efficient way is one of the bigger challenges with GraphQL, it's not all sunshine and roses. GraphQL is not a silver bullet but it makes life a lot easier

## Building blocks

A GraphQL Server consists of the following:

*   **a schema**, the schema defines our entities but also what we can query or call a mutation on
*   **resolvers**, resolver functions talks to a 3rd party API or our database and ends up returning data back to our user

## Install dependencies

Let’s start off by installing our needed dependencies. We need the following:

*   **express**, to create our web server
*   **graphql**, to install graphql, our core lib that enables us to leverage graphql
*   **express-graphql**, this library enables us to bind together graphql and express

## Express + graphql (only)

Let’s start off by only installing `graphql` and `express` to understand what that gives us:


    npm install express graphql

Thereafter let’s create an `express` HTTP server, like so:


    // schema.mjs

    import {
      graphql,
      GraphQLSchema,
      GraphQLObjectType,
      GraphQLString,
      GraphQLList
    } from "graphql";
    let humanType = new GraphQLObjectType({
      name: "Human",
      fields: () => ({
        id: { type: GraphQLString },
        description: { type: GraphQLString },
        name: { type: GraphQLString }
      })
    });
    import people from "./data/people";
    let schema = new GraphQLSchema({
      query: new GraphQLObjectType({
      name: "RootQueryType",
      fields: {
        hello: {
          type: GraphQLString,
          resolve() {
            return "world";
          }
        },
        person: {
          type: humanType,
          resolve() {
            return people[0];
          }
        },
        people: {
          type: new GraphQLList(humanType),
          resolve() {
            return people;
          }
        }
      }
    })
    });

    export { graphql };
    export default schema;

This is a pretty simple schema that declares `hello`, `person` and `people` as queryable keywords and it also creates `humanType` as a custom type.

A short comment on the file ending `.mjs`. What we are doing here is to leverage experimental support for `ESM/EcmaScript` modules. The way they are currently implemented in NodeJS forces us to have a file ending of `.mjs`.

Next up is the app itself that is just a basic express application looking like this:

    // app.mjs
    import express from "express";
    const app = express();
    const port = 3000;
    import schema, { graphql } from "./schema";

    app.get("/", (req, res) => {
      let query = `{ hello, person { name }, people { name, description } }`;
      graphql(schema, query).then(result => {
        res.json(result);
      });
    });
    app.listen(port, () => console.log(`Example app listening on port port!`));

Above we are declaring a default route by calling:


    app.get("/", (req, res) => {
    });

Then we add the `graphql` part by invoking it with parameters `schema` and `query`, like so:

    graphql(schema, query).then(result => {
      res.json(result);
    });

As we can see above invoking `graphql` means we get a promise back and on the `then()` callback we are able to see the result from our query. All together we can see how `graphql` and `express` can interact.

Lastly, to run this we need to specify a `start` command in our `package.json` file that invokes the experimental support for ESM modules. It needs to look like so:


    // excerpt from package.json
    "start": "node — experimental-modules app.mjs"

### Adding express-graphql

We just showed how we can use `express` and `graphql` and create a REST api, but we can do this better by adding `express-graphql`, so let’s do that as our next thing:

    npm install express-graphql

Let’s do something else first, for ourselves, namely, use the `buildSchema()` method and set up a schema that way, like so:

    var { buildSchema } = require("graphql");
    var schema = buildSchema(`
      type Product {
        name: String,
        id: Int
      },
      type Query {
        hello: String,
        products: [Product]
      }
    `);

Above we can see that we define the custom type `Product` and we also define our queries being `hello` and `products`.

We also need some resolver functions with that which we define next:

    const getProducts = () => {
      return Promise.resolve([{
        title: 'Movie'
      }]);
    }  

    var root = {
      hello: () => {
        return "Hello world!";
      },
      products: () => {
        return getProducts();
      }
    };

Lastly, we are able to clean up our code a bit so our code for starting our server looks like this now:

    var graphqlHTTP = require("express-graphql");
    app.use(
      '/graphql',
      graphqlHTTP({
        schema: schema,
        rootValue: root,
        graphiql: true
      })
    );

That’s it, we don’t actually need to define any routes but we leave that up to graphql. We can see that `graphqlHTTP()` is a function that we get from `express-graphql`

Now we have all the bits in place.

### **Graphiql**

When we called our `graphqlHTTP()` function we provided it with a configuration object that had the following properties set:

*   **schema**, our GraphQL schema
*   **rootValue**, our resolver functions
*   **graphiql**, a boolean stating whether to use `graphiql`, we want that so we pass `true` here

Next step is to try out `graphiql` which we do by navigating to `http://localhost:4000/graphql` and voila, this is what you should be seeing:

![](https://cdn-images-1.medium.com/max/1600/1*X8nQBrmfPu4nmbzPjUNGhA.png)

Ok great, a visual interface, now what?

Well, now you can start creating Graphql queries. To know what to query for, have a look at what you defined in the schema.

We expect that we will be able to query for `hello` and `products` as we set those up in our schema. So let’s do that:

![](https://cdn-images-1.medium.com/max/1600/1*X_D6Hbp9JO71SHea0M03fQ.png)

Ok then, you should be seeing the above by hitting the `play` icon. As you can see this is a very useful tool to debug your queries but it can also be used to debug your mutations.

### Parameterized query

Let’s try to write a query with parameters in graphiql:

![](https://cdn-images-1.medium.com/max/1600/1*eD5LVw96Aq9kmzaser0yMg.png)

Above we can see how we define our query by using the keyword `query`. Thereafter we give it a name `Query` followed by a parenthesis. Within the parenthesis, we have the input parameter that we denote with `<div class="theme-default-content content__default" character. In this case, we call our parameter `id`, which means its full name is `$id`. Let’s look at what we got:

    query Query($id: Int!) {
      // our actual query
    }

Now it’s time to write our actual query, so let’s do that next:

    product(id: $id) {
      name
    }

As you can see we use the `$id` from our query construction and the full result looks like this:

    query Query($id: Int!) {
      product(id: $id) {
        name
      }
    }

> NOTE: at the bottom of the screen we have a section called `query variables` that lets us define variables we want to use as input. So when we define something as `$name_of_variable` it will look into the `query variables` section

**calling a Mutation**

To invoke a mutation we need the `mutation` keyword. Let’s create our mutation invocation next:

    mutation MyMutation {
      addProduct(name: "product", description: "description of a product") 
    }

> NOTE: if we return a complex object we need to select what columns we want, like so



    mutation MyMutation {
      addProduct(name: "product", description: "description of a product"){ 
        col1, 
        col2 
      }
    }


## Summary

To build an API we have used the NPM libraries `express`, `graphql`. However, by adding `express-graphql` we have gained access to a visual environment called `graphiql` that enables us to pose queries and run mutations so we can verify that our API works as intended