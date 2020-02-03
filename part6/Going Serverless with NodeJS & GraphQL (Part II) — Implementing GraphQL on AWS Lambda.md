# Going Serverless with NodeJS & GraphQL (Part II) — Implementing GraphQL on AWS Lambda

Before you read this artcle, note that there is the initial article on cloud-formation templates [here](/going-serverless-with-nodejs-graphql-5b34f5d280f4).

![](https://miro.medium.com/max/380/1*yyyTyXq4mnpdsXLnJHjDzg.png)

Continuing on from where we left off on the previous [article](/going-serverless-with-nodejs-graphql-5b34f5d280f4). We will be looking at integrating GraphQL into the Lambda Function we created on cloud formation template. But first, let’s learn a little bit about GraphQL.


![](https://miro.medium.com/max/1024/1*2sNJC8zoSY1iXIiqGhz2Zw.jpeg)

> _GraphQL_ is a query language _and_ runtime that we can use to build and expose APIs as a strongly-typed schema instead of hundreds of REST endpoints. Your clients see the schema. They write a query for what they want. They send it over and get back exactly the data they asked for and nothing more. { Source: [_A Guide to GraphQL in Plain English_](https://medium.freecodecamp.org/a-beginners-guide-to-graphql-60e43b0a41f5) _by_ [_Luis Aguilar_](https://medium.freecodecamp.org/@ldiego08?source=post_header_lockup) _}_

![](https://miro.medium.com/max/1038/1*i4ICY97Kg0eOks82ibUQwA.jpeg)

GraphQL from a birds eye : [source](https://www.youtube.com/watch?v=Tpf9kVE2AY8)

As the above image shows, GraphQL represents a dataset branching from a single endpoint with a map of pointers/references to more related data. Full flexibility is given to the client to query data as it wants.

Back to our Lambda function — Start off by creating a `handler.js` or in my case I will be creating `handler.ts` (TypeScript file) which will compile to `handler.js` with the help of [Webpack](https://webpack.js.org/) and [Babel 7](https://babeljs.io/docs/en/index.html).(You can find the Webpack config and plugins used [here](https://gitlab.com/DasithKuruppu/serverlessgraphql/blob/master/webpack.config.js)). The `handler.ts` would contain the following function:

```
import schema from "./schemas/index";
import { graphql } from "graphql";
import { APIGatewayProxyEvent } from "aws-lambda";
// Export a function which would be hooked up to the the λ node/ nodes as specified on serverless.yml template
export async function queryEvents(
  event: APIGatewayProxyEvent,
  // context: Context,
) {
  const parsedRequestBody = (event && event.body) ? JSON.parse(event.body) : {};
  try {
    const graphQLResult = await graphql(schema,
      parsedRequestBody.query,
      null,
      null,
      parsedRequestBody.variables,
      parsedRequestBody.operationName);

    return { statusCode: 200, body: JSON.stringify(graphQLResult) };
  } catch (error) {
    throw error;
  }
}
```

Here we export an async function `queryEvents` which has access to the standard arguments provided by API gateway i.e`event, context, and callback` In our case we just need `event` and `context`. We won’t need the `callback` function because we will be using `async/await`.

We are always expecting a GraphQL query to this endpoint / path, so we will parse the `event.body` to convert JSON to a JavaScript Object.

We will be using the `graphql` package which can be installed using `npm install graphql`. We import `graphql` into our Lambda function and invoke it with the parsed query.

GraphQL expects a schema. I will be using a simple schema like this to get started:

```
import {
  GraphQLSchema,
  GraphQLObjectType,
  GraphQLString,
  GraphQLList,
  GraphQLNonNull,
  GraphQLBoolean,
} from "graphql";

import {
  GraphQLDateTime,
} from "graphql-iso-date";

import { IEvent } from "../resolvers/events/typings";
import addEvent from "../resolvers/events/create";
import viewEvent from "../resolvers/events/view";
import listEvents from "../resolvers/events/list";
import removeEvent from "../resolvers/events/remove";

const eventType = new GraphQLObjectType({
  name: "Event",
  fields: {
    id: { type: new GraphQLNonNull(GraphQLString) },
    name: { type: new GraphQLNonNull(GraphQLString) },
    description: { type: new GraphQLNonNull(GraphQLString) },
    addedAt: { type: new GraphQLNonNull(GraphQLDateTime) },
  },
});

const schema = new GraphQLSchema({
  query: new GraphQLObjectType({
    name: "Query",
    fields: {
      listEvents: {
        type: new GraphQLList(eventType),
        resolve: (parent ) => {
          return listEvents();
        },
      },
      viewEvent: {
        args: {
          id: { type: new GraphQLNonNull(GraphQLString) },
        },
        type: eventType,
        resolve: (parent, args: { id: string }) => {
          return viewEvent(args.id);
        },
      },
    },
  }),

  mutation: new GraphQLObjectType({
    name: "Mutation",
    fields: {
      createEvent: {
        args: {
          name: { type: new GraphQLNonNull(GraphQLString) },
          description: { type: new GraphQLNonNull(GraphQLString) },
        },
        type: eventType,
        resolve: (parent, args: IEvent) => {
          return addEvent(args);
        },
      },
      removeProduct: {
        args: {
          id: { type: new GraphQLNonNull(GraphQLString) },
        },
        type: GraphQLBoolean,
        resolve: (parent, args: { id: string }) => {
          return removeEvent(args.id);
        },
      },
    },
  }),
});
export default schema;
```

The schema above specifies a few queries and mutations. A query in GraphQL is used for reading data, and a mutation is used to change data i.e usually on a persistent layer. We also provide the fields the data would contain and what it would resolve to. In our case the resolve calls a function which in turn makes an async call to the database (DynamoDb) to read or mutate data. For example the `viewEvent(args.id)` calls the below function

```
import { DynamoDB } from "aws-sdk";
const dynamoDb = new DynamoDB.DocumentClient();

export default (id: string) => {
  const params = {
    TableName: process.env.TABLE_NAME,
    Key: { id },
  };
  return dynamoDb
    .get(params)
    .promise()
    .then((GetEvents) => GetEvents.Item);
};
```

Once all this is set (you can find the rest of the resolvers/events [here](https://gitlab.com/DasithKuruppu/serverlessgraphql/tree/master/resolvers/events)) you can then start creating the resources in the template and deploying code to the Lamda function by running:

`serverless deploy -v — stage=default`

> Note for this to work you need to follow the steps [part I](/going-serverless-with-nodejs-graphql-5b34f5d280f4) **or** cloned and followed readme on [Serverless Boilerplate](https://gitlab.com/DasithKuruppu/serverlessgraphql)

This deploys the relevant resources needed and finally gives you a URL to access the lambda function see below.

![](https://miro.medium.com/max/2068/1*tGarXiKYlGHtro-jrLY7Jg.png)

Now you can install [GraphQL Playground](https://github.com/prisma/graphql-playground/releases) and use it to run queries against this URL.

![](https://miro.medium.com/max/2370/1*bjFmf9HI2KPmVmM9huZbFw.png)

Select `url endpoint` and then paste the copied URL from the terminal. Open the URL to query against the endpoint. Let’s try by running a mutate query to create an Event:

```
mutation {
  createEvent(name: "Random event", description: "Random Description") {
    id
    name
    description
  }
}
```

which should give you an output like the following …

![](https://miro.medium.com/max/3814/1*Xjp6sf0nhs0bOCCW9pQjEg.png)

And that should be enough to get you started on GraphQL. The repository/boilerplate below will help you to get started right away. It provides you with serverless yml templates, GraphQL API, Typescript / webpack config, Jest unit testing capabilities, serverless offline, debugging capabilities on VS-Code & CI/CD pipelines on [git-lab](https://about.gitlab.com/) straight away !