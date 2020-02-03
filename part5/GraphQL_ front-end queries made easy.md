# GraphQL: front-end queries made easy

![GraphQL: front-end queries made easy](https://cdn-media-1.freecodecamp.org/images/1*49DDRZhUWvVnH-QNHuSUSw.png)
by Rasheed Bustamam

If you’ve been reading about the latest trends in front-end development, you may have heard some buzz about something called GraphQL. In this article, we will cover what GraphQL is (and isn’t), some best practices behind GraphQL, and most importantly, why it makes your job as a front-end developer much easier.

_Note: This is not meant to be a GraphQL tutorial. This article is merely meant to describe why GraphQL makes your job as a front-end developer easier._

### What is GraphQL?

GraphQL is a query language created by Facebook. It allows you, the front-end developer, to write queries that have the exact data shape you want.

It’s important to note that despite the name “GraphQL,” it is _not_ meant to be a database language. In fact, I’ve used GraphQL with many databases, including SQL, MongoDB, and even a REST API connected to a database (and I don’t even know what the database language was written in).

Instead, GraphQL on the server allows you to define how you want to fetch your data. Then, on the client, you can execute a query that has the exact data shape you want, and the data you get back will have the same shape as the query.

For example:

The response you get back will look something similar to this:

There will be other metadata in the response, but you can already see how powerful GraphQL is — no need to normalize any of your data on the client. The work is pushed to the server.

Since this article is more oriented towards front-end developers, I won’t get into too much detail about how GraphQL works on the server. However, what makes the “magic” happen is the GraphQL schema, which allows you to define types that are used across server and client. Here’s an example schema for a hypothetical app:

Just by looking at the schema, you can already tell what the shape of your data will be, and what types are expected. This allows you, the front-end developer, to know several important properties of a query, such as if it takes arguments, if it’s a list or just an “entity,” and if it’s a specific entity, what fields that entity has (in this case, entity `User` has fields `name` and `email`).

In the case of the `user` query, we can see that it takes in an argument called `username`, which has a type of `String`. The `!` denotes that the field is required, so if it is not supplied, an error will be thrown.

As a front-end developer, you probably won’t be working too much with the schema, but it will serve as an important document that allows you to know what queries you are allowed to make.

Compare that to other API standards, such as Swagger for REST APIs — you have to trust that whoever wrote documentation wrote it really well, with all edge cases and types documented. Swagger doesn’t exactly “enforce” type checking for different fields and such, so you can have a valid Swagger YAML file that is still incredibly difficult to navigate.

However, any valid GraphQL schema in itself will be immensely helpful for anyone to know what sorts of data they’re dealing with, even if it isn’t properly commented out and documented.

This isn’t meant to dig at Swagger or to say it’s terrible — used correctly, Swagger can be very useful. But that’s the caveat — it needs to be used correctly, and since developers tend to move extremely fast, proper documentation can often take a back seat to building new and exciting features/APIs.

### How to use GraphQL on the client

This is a fun one. On the client-side, there are many ways to use GraphQL on the client.

One of the most popular ways to use GraphQL is using a library called [**apollo-client**](https://github.com/apollographql/apollo-client). Apollo Client can interface with [React](https://www.apollographql.com/docs/react/), [Vue](https://github.com/akryum/vue-apollo), [Angular](https://www.apollographql.com/docs/angular/), and more.

Now, Apollo Client recently updated to the 2.0 release. It is absolutely _not_ backwards compatible with the 1.0 release, with many packages changing names and entire APIs changing. I’ve been slowly familiarizing myself with the 2.0 release, but there are still some things I was able to do in 1.0 that I can no longer do in 2.0, such as Redux integration in my React apps. Because of this, I would consider 1.0 and 2.0 to be entirely different ways to use GraphQL on the client.

However, the overall concept is the same: wrap your entire app in an Apollo Provider (similar to how you would do with Redux), and now all of your components have access to the client, and can write queries and mutations to the server.

Apollo Client does a lot of cool “behind-the-scenes” things that you expect _should_ be standard but apparently are not. One example is “batching” queries. If I load a component that loads two different queries, the default is to send two different requests. However, Apollo Client has the option to “batch” these queries, which puts both of those queries into a single request and sends that up to the server, saving some HTTP requests.

Apollo Client also has a very robust caching feature, which makes components fetch from the cache first. Then it actually issues a request if the cache is stale (usually 100ms old, but can be configured).

Here’s an example of instantiating an Apollo Client, and issuing a query:

This doesn’t even use React. If you were to implement React, then you could actually attach the query to the React Component so that it receives the query data as props.

The other way to use GraphQL on the client is by using [**Relay**](https://facebook.github.io/relay/), which only works with React. So sorry Vue devs, you can’t use Relay.

I haven’t used Relay too much, but it definitely has a steeper learning curve than Apollo. It seems like you have to “DIY” for a lot of things, such as caching and even schema implementation. You can take a look at some of the [examples here](https://github.com/relayjs/relay-examples) to get an overview of how Relay works, and how it’s similar to and different from Apollo.

Once you get Relay set up, then it actually works very similarly to how Apollo Client and react-apollo work together, in that it sends in the data as props to the component.

### Wrapping it up

I hope this article was useful for you in deciding if you should use GraphQL or not. For me, just knowing the shape of the data coming in makes my front-end work immensely easier. And if the data is not coming in correctly, then I change it in the schema in the back-end, and update any necessary server-side code.

If this article has piqued your interest in learning GraphQL, I suggest taking a deeper look at the official GraphQL tutorial: [How to GraphQL](https://www.howtographql.com/).

Have fun querying!