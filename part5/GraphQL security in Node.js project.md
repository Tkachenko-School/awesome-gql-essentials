# GraphQL security in Node.js project

![](https://miro.medium.com/max/1350/1*pR4PYnwlPEUWiPcPq8CxuQ.jpeg)

## Introduction

[GraphQL](https://graphql.org/) is a powerful tool, but with great power come great security risks. One of the biggest advantages of GraphQL is that you are able get data across multiple resources within a single request. This allows the potential attacker to make complex queries that quickly result in resource exhaustion. In this brief article we will go through some tips to minimize these risks and secure your GraphQL schema against potential attackers.

> [Checkout the upcoming **free** **GraphQL language course** with in browser exercices](https://graphqlmastery.com/course/graphql-language)

![](https://miro.medium.com/max/545/0*K2kl-FlLs0dvGqXa.gif)

From upcoming [GraphQL Language course](https://graphqlmastery.com/course/graphql-language)

## Use https and do not forget about https redirect

I would not say this issue is [GraphQL](https://graphql.org/learn/) specific, but nearly all websites should use https. Moreover, you are communicating with the server in a more secure way. This will also improve your SEO. We often find that some developers forget to add https redirect or an hts header to your server. Then, if you access [http://graphqlmastery.com](https://graphqlmastery.com/) you will be redirected to the https version and then communicate with the using http protocol by accident. If you use express it is also good practice from a security point of view to add [helmet](https://github.com/helmetjs/helmet) middleware to existing server. This library will adjust headers in each request to be more secure.

## Disable [introspection](https://graphqlmastery.com/blog/graphql-introspection-and-introspection-queries) in production

If you are familiar with the tools like GraphiQL, you are maybe wondering, how you can know everything about the schema. In GraphQL there is an option to execute the so-called introspection queries of the schema. You can use this tool to know basically everything about the type system of the schema including, what you can query for, available mutations, etc. If you are in a development environment, it is definitely useful to allow introspection for various purposes. In production, however, it can leak important information for potential attackers or it will just reveal information about your new feature, which is not implemented on the fronted. If you want to solve this issue you can use the library called GraphQL [GraphQL Disable Introspection](https://github.com/helfer/graphql-disable-introspection). It allows you to add validation rules that disable introspection. To disable the introspection for everyone is sometimes a bit limited. Therefore it is much better to add introspection on per request bases or to enable introspection only for certain scopes.

## Masking errors

When it comes to error handling it is helpful to have a clearly defined method to deal with errors in your GraphQL project. However, it is important to mask every error that users are not allowed to view. For example, if you use an SQL builder such as [knex.js](https://knexjs.org/), you can then reveal information about your database schema and leak important facts about the project structure to attacker. The proper way to do this is to add reactive middleware to your resolvers which enhances each resolver with masking functionality. A good library for that is [apollo resolvers](https://github.com/thebigredgeek/apollo-resolvers) with a combination of [apollo errors](https://github.com/thebigredgeek/apollo-errors). With this design pattern you will not only implement masking errors, but you can also add additional permission logic. The other possibility is to add error masking to a format error callback in your [apollo server.](https://github.com/apollographql/apollo-server) and use available errors in apollo server.

## Resource exhaustion prevention

I think the biggest issue in GraphQL (especially if you want to open the schema to the public) comes with its biggest advantage, and that is the ability to query for various sources with a one single request. However, there are certain concerns about this feature. The issue is that potential attackers can easily call complex queries, which may be extremely expensive for your server and network. We can reduce the load on the database a lot by batching and caching with [Data Loader](https://github.com/facebook/dataloader). Load on the network, however, can not be reduced easily and has to be restricted. There are various ways to limit capabilities of the attacker to execute malicious queries. In my opinion, the most important and the most useful methods are the following:

*   **Rejecting based on query complexity (cost analysis)** great for public schema, but needed even for queries behind authorization. A great library for this use case is [graphql-cost-analysis](https://github.com/pa-bru/graphql-cost-analysis) as it also provides different cost analysis rules based on the query and not just for the whole schema.
*   **Amount limiting** restrict the amount of objects someone is able to fetch from the database. Instead of fetching every object, it’s better to use cursor based pagination.
*   **Depth limiting** block recursive queries, which are too costly. Usually limiting the amount to depth 10 is good enough.

There are many more methods you can implement, but the combination of these three you will cover most cases of malicious queries. None of these methods will solve the problem for every query. Therefore we need to implement a combination of these methods. We will include a detailed explanation with implementation in future articles, so be sure to subscribe.

## Use npm audit in your CI

One of the biggest security issues in your Node.js project is that you can accidentally use a malicious package or a package with security holes. The danger exists not only for lesser known npm packages as described in [this article](https://hackernoon.com/im-harvesting-credit-card-numbers-and-passwords-from-your-site-here-s-how-9a8cb347c5b5), but also for the packages with a large user base. Let’s take the example of the latest incident, which affected the package [eslint-scope](https://github.com/eslint/eslint-scope), which in turn is dependent on some widely used packages like [babel-eslint](https://github.com/babel/babel-eslint) and [webpack](https://github.com/webpack/webpack), see [postmortem](https://eslint.org/blog/2018/07/postmortem-for-malicious-package-publishes). In this incident the the credentials of one of the the contributors were compromised, and then the new version of the packages with malicious code was published. You will never be able to defend yourself fully if you use some external packages, but you can significantly decrease the risk by using [npm audit](https://docs.npmjs.com/getting-started/running-a-security-audit) in your continuous integration pipeline.

## Summary

The list definitely does not end here. This is only a small subset of security concerns that you need to consider when deploying your GraphQL app to production. To really dive deeper into these topics with the practical code snippets on implementing all security concerns, you can subscribe to our GraphQL newsletter, where you will receive info about the newest articles on GraphQL Mastery and also updates on our upcoming GraphQL courses. Meanwhile, you can take a look at our free GraphQL language course.