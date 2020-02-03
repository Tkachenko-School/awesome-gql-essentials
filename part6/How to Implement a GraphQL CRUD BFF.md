# How to Implement a GraphQL CRUD BFF

## _Learn how to implement a GraphQL CRUD BFF in this tutorial by John Gilbert, the author of_ [_JavaScript CloudNative Development Cookbook_](https://www.packtpub.com/application-development/javascript-cloud-native-development-cookbook)_._

The BFF pattern accelerates innovation because the team that implements the frontend also owns and implements the backend service that supports the frontend. This enables teams to be self-sufficient and unencumbered by competing demands for a shared backend service. In this article, you will create a CRUD BFF service that supports data at the beginning of its life cycle.

The single responsibility of this service is authoring data for a specific bounded context. It leverages **database-first** Event Sourcing to publish domain events to downstream services. The service exposes a GraphQL-based API.

![](https://miro.medium.com/max/1000/1*55Tp4JdXkjmDk06cofn6qg.png)



# Getting ready

Before starting this recipe, you will need an AWS Kinesis Stream.

# How to do it…

1.  Create the project from the following template:

```
$ sls create - template - url https: //github.com/danteinc/js-cloud-native-cookbook/tree/master/ch3/bff-graphql-crud - path cncb-bff-graphql-crud
```

2\. Navigate to the `cncb-bff-graphql-crud` directory with `cd cncb-bff-graphql-crud`.

3\. Then, review the file named `serverless.yml` with the following content:

```
service: cncb - bff - graphql - crud
provider:
    name: aws
runtime: nodejs8 .10
iamRoleStatements: …
    functions:
    graphql:
    handler: handler.graphql
events:
    -http:
    path: graphql
method: post
cors: true
environment:
    TABLE_NAME:
    Ref: Table
graphiql:
    handler: handler.graphiql…
trigger:
    handler: handler.trigger
events:
    -stream:
    type: dynamodb…
resources:
    Resources:
    Table:
    Type: AWS::DynamoDB::Table…
 ```

4\. Review the file named `./schema/thing/typedefs.js` with the following content:

```
module.exports = `
  type Thing {
    id: String!
    name: String
    description: String
  }
  type ThingConnection {
    items: [Thing!]!
    cursor: String
  }
  extend type Query {
thing(id: String!): Thing
things(name: String, limit: Int, cursor: String): ThingConnection
  }
  input ThingInput {
    id: String
    name: String!
    description: String
  }
  extend type Mutation {
saveThing(
      input: ThingInput
    ): Thing
deleteThing(
      id: ID!
    ): Thing
  }
`;
```

5\. Review the file named `./schema/thing/resolvers.js` with the following content:

```
module.exports = {
Query: {
thing(_, { id }, ctx) {
      return ctx.models.Thing.getById(id);
    },
things(_, { name, limit, cursor }, ctx) {
      return ctx.models.Thing.queryByName(name, limit, cursor);
    },
  },
Mutation: {
saveThing: (_, { input }, ctx) => {
      return ctx.models.Thing.save(input.id, input);
    },
deleteThing: (_, args, ctx) => {
      return ctx.models.Thing.delete(args.id);
    },
  },
};
```

6\. Review the file named `handler.js` with the following content:

```
...
const { graphqlLambda, graphiqlLambda } = require('apollo-server-lambda');
const schema = require('./schema');
const Connector = require('./lib/connector');
const { Thing } = require('./schema/thing');

module.exports.graphql = (event, context, cb) => {
graphqlLambda(
    (event, context) => {
      return {
schema, context: { models: {
            Thing: new Thing( new Connector(process.env.TABLE_NAME) )
          } }
      };
    }
  )(event, context, (error, output) => {
    cb(error, ...);
  });
};
. . .
```

7\. Install the dependencies with npm install.

8\. Run the tests with `npm test — -s $MY_STAGE`.

9\. Review the contents generated in the `.serverless` directory.

10\. Deploy the stack:

```
$ npm run dp:lcl -- -s $MY_STAGE

> cncb-bff-graphql-crud@1.0.0 dp:lcl <path-to-your-workspace>/cncb-bff-graphql-crud
> sls deploy -v -r us-east-1 "-s" "john"

Serverless: Packaging service...
...
Serverless: Stack update finished...
...
endpoints:
  POST - https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/john/graphql
  GET - https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/john/graphiql
functions:
  graphql: cncb-bff-graphql-crud-john-graphql
  graphiql: cncb-bff-graphql-crud-john-graphiql
  trigger: cncb-bff-graphql-crud-john-trigger

Stack Outputs
...
ServiceEndpoint: https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/john
...
```

11\. Review the stack in the AWS Console.

12\. Invoke the function with the following curl commands:

```
$curl -X POST -H 'Content-Type: application/json' -d '{"query":"mutation { saveThing(input: { id: \"33333333-1111-1111-1111-000000000000\", name: \"thing1\", description: \"This is thing one of two.\" }) { id } }"}' https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/$MY_STAGE/graphql | json_pp

{
   "data" : {
      "saveThing" : {
         "id" : "33333333-1111-1111-1111-000000000000"
      }
   }
}

$ curl -X POST -H 'Content-Type: application/json' -d '{"query":"mutation { saveThing(input: { id: \"33333333-1111-1111-2222-000000000000\", name: \"thing2\", description: \"This is thing two of two.\" }) { id } }"}' https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/$MY_STAGE/graphql | json_pp

{
   "data" : {
      "saveThing" : {
         "id" : "33333333-1111-1111-2222-000000000000"
      }
   }
}

$curl -X POST -H 'Content-Type: application/json' -d '{"query":"query { thing(id: \"33333333-1111-1111-1111-000000000000\") { id name description }}"}' https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/$MY_STAGE/graphql | json_pp

{
   "data" : {
      "thing" : {
         "description" : "This is thing one of two.",
         "id" : "33333333-1111-1111-1111-000000000000",
         "name" : "thing1"
      }
   }
}

$curl -X POST -H 'Content-Type: application/json' -d '{"query":"query { things(name: \"thing\") { items { id name } cursor }}"}' https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/$MY_STAGE/graphql | json_pp

{
   "data" : {
      "things" : {
         "items" : [
            {
               "id" : "33333333-1111-1111-1111-000000000000",
               "name" : "thing1"
            },
            {
               "name" : "thing2",
               "id" : "33333333-1111-1111-2222-000000000000"
            }
         ],
         "cursor" : null
      }
   }
}

$curl -X POST -H 'Content-Type: application/json' -d '{"query":"query { things(name: \"thing\", limit: 1) { items { id name } cursor }}"}' https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/$MY_STAGE/graphql | json_pp

{
   "data" : {
      "things" : {
         "items" : [
            {
               "id" : "33333333-1111-1111-1111-000000000000",
               "name" : "thing1"
            }
         ],
         "cursor" : "eyJpZCI6IjMzMzMzMzMzLTExMTEtMTExMS0xMTExLTAwMDAwMDAwMDAwMCJ9"
      }
   }
}

$curl -X POST -H 'Content-Type: application/json' -d '{"query":"query { things(name: \"thing\", limit: 1, cursor:\"CURSOR VALUE FROM PREVIOUS RESPONSE\") { items { id name } cursor }}"}' https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/$MY_STAGE/graphql | json_pp

{
   "data" : {
      "things" : {
         "items" : [
            {
               "id" : "33333333-1111-1111-2222-000000000000",
               "name" : "thing2"
            }
         ],
         "cursor" : "eyJpZCI6IjMzMzMzMzMzLTExMTEtMTExMS0yMjIyLTAwMDAwMDAwMDAwMCJ9"
      }
   }
}

$ curl -X POST -H 'Content-Type: application/json' -d '{"query":"mutation { deleteThing( id: \"33333333-1111-1111-1111-000000000000\" ) { id } }"}' https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/$MY_STAGE/graphql | json_pp

{
   "data" : {
      "deleteThing" : {
         "id" : "33333333-1111-1111-1111-000000000000"
      }
   }
}

$curl -X POST -H 'Content-Type: application/json' -d '{"query":"mutation { deleteThing( id: \"33333333-1111-1111-2222-000000000000\" ) { id } }"}' https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/$MY_STAGE/graphql | json_pp

{
   "data" : {
      "deleteThing" : {
         "id" : "33333333-1111-1111-2222-000000000000"
      }
   }
}

$curl -X POST -H 'Content-Type: application/json' -d '{"query":"query { things(name: \"thing\") { items { id } }}"}' https://ac0n4oyzm6.execute-api.us-east-1.amazonaws.com/$MY_STAGE/graphql | json_pp

{
   "data" : {
      "things" : {
         "items" : []
      }
   }
}
```

13\. Perform the same mutations and queries using GraphQL by using the endpoint output during deployment:

![](https://miro.medium.com/max/1428/1*pOfKmJs_asyadKBLFEpgXA.png)

14\. Take a look at the `trigger` function logs:

```
$ sls logs -f trigger -r us-east-1 -s $MY_STAGE
```

15\. Review the events collected in the data lake bucket.

16\. Remove the stack once you are finished with `npm run rm:lcl — -s $MY_STAGE`.

# How it works…

**GraphQL** is becoming increasingly popular because of the flexibility of the resulting API and the power of client libraries, such as the Apollo Client. We implement a single `graphql` function to support our API and then add the necessary functionality through the `schema`, `resolvers`, `models`, and `connectors`.

The GraphQL schema is where we define our `types`, `queries`, and `mutations`. In this tutorial, we can query `thing` types by ID and by name, and `save` and `delete`. The `resolvers` map the GraphQL requests to the `model` objects that encapsulate the business logic.

The `models`, in turn, talk to the `connectors` that encapsulate the details of the database API. The `models` and `connectors` are registered with the `schema` in the `handler` function with a very simple but effective form of constructor-based dependency injection.

We don’t use dependency injection very often in cloud-native because the functions are so small and focused that it is overkill and can impede performance. With GraphQL, this simple form is very effective for facilitating testing. The `Graphiql` tool is very useful for exposing the self-documenting nature of APIs.

The single responsibility of this service is authoring data and publishing the events, using database-first Event Sourcing, for a specific bounded context. The code within the service follows a very repeatable coding convention of `types`, `resolvers`, `models`, `connectors`, and `triggers`. As such, it is very easy to reason about the correctness of the code, even as the number of business domains in the service increases.

That’s why it is reasonable to have a larger number of domains in a single authoring BFF services, so long as the domains are cohesive, part of the same bounded context, and authored by a consistent group of users.

_We hope you found this tutorial helpful. If you want to learn more about Cloud Native, you can read_ [_JavaScript Cloud Native Development Cookbook_](https://www.amazon.com/JavaScript-Cloud-Native-Development-Cookbook/dp/1788470419)_. This book helps you learn the major concepts of cloud-native development faster by taking a recipe-based approach, where you can try out different solutions to understand the concepts._