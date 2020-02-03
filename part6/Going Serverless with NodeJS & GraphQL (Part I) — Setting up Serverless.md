# Going Serverless with NodeJS & GraphQL (Part I) — Setting up Serverless

For the past few months, I have been interested in serverless on AWS using lambda functions and Cloud-formation templates to provide resources. I have decided to share what I have learned and created which I think would help anyone else that wants to dive into Serverless/cloud-formation on AWS with a head start :)

This first article (part 1) will show how to initialize Serverless and the next article ([part 2](/going-serverless-with-nodejs-graphql-part-ii-1810445028a4)) will show how to implement GraphQL on top of it.

Below you will find my initial plan / architecture. It is useful for generic apps, and for my use-case, I will be building a booking application.

![](https://miro.medium.com/max/3840/1*luzrrxaE39RRNO96vHFpAA.png)

Initially, I started on the [serverless](https://serverless.com/) website went through their basics, after a few hours of reading and research. I could put together template files that resembled the architecture above. Below is a gist of my first template file.

```
service: aws-nodejs
plugins:
custom:
  # Our stage is based on what is passed in when running serverless
  # commands. Or fallsback to what we have set in the provider section.
  stage: ${opt:stage, self:provider.stage}
  # Set the table name here so we can use it while testing locally
  tableName: ${self:custom.stage}-events
  # Set our DynamoDB throughput for prod and all other non-prod stages.
  tableThroughputs:
    prod: 1 #more throughput on production env 
    default: 1
  tableThroughput: ${self:custom.tableThroughputs.${self:custom.stage}, self:custom.tableThroughputs.default}
  # Load our secret environment variables based on the current stage.
  # Fallback to default if it is not in prod.
  environment: ${file(env.yml):${self:custom.stage}, file(env.yml):default}
provider:
  name: aws
  runtime: nodejs8.10
  stage: default
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:DescribeTable
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource:
        Fn::Join:
          - ""
          - - "arn:aws:dynamodb:*:*:table/"
            - Ref: EventsGqlDynamoDbTable
functions:
  queryEvents:
    handler: handler.queryEvents
    events:
      - http:
          path: events
          method: post
          cors: true
    environment:
      TABLE_NAME: ${self:custom.tableName}
resources:
  # DynamoDB
  - ${file(resources/dynamodb.yml)}
  # S3
  - ${file(resources/s3-bucket.yml)}
  # Cognito
  - ${file(resources/cognito-user-pool.yml)}
  - ${file(resources/cognito-identity-pool.yml)}
```

Don’t worry about the externally linked files (under resources for now). The idea of this template is to create resources on [AWS](https://aws.amazon.com/), such as a Lambda function which can execute actions on [DynamoDB](https://aws.amazon.com/dynamodb/) (DaaS provided by AWS).

Explicit permissions need to be given to Lambda functions, and in this instance, we are providing them the ability to interact with DynamoDB by specifying the possible actions it can take on it. The resources it can perform these interactions on must be explicitly declared as well — in this case, a DynamoDB table with the Reference `EventsGqlDynamoDbTable`. The below snapshot shows all the permissions that I have mentioned.

```
provider:
  name: aws
  runtime: nodejs8.10
  stage: default
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:DescribeTable
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource:
        Fn::Join:
          - ""
          - - "arn:aws:dynamodb:*:*:table/"
            - Ref: EventsGqlDynamoDbTable
```

Finally, we provide a function that gets triggered based on an HTTP POST to a specific path on AWS API Gateway. In this instance, the route is `/events`. This single endpoint would be the place where we will connect up our GraphQL API later in the tutorial.

```
functions:
  queryEvents:
    handler: handler.queryEvents
    events:
      - http:
          path: events
          method: post
          cors: true
    environment:
      TABLE_NAME: ${self:custom.tableName}
```

Then we provide a set of references to the resources that we will be actually using:

```
resources:
  # DynamoDB
  - ${file(resources/dynamodb.yml)}
  # S3
  - ${file(resources/s3-bucket.yml)}
```

Where the YML referencing DynamoDB is:

```
Resources:
  EventsGqlDynamoDbTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: ${self:custom.tableName}
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
    # Set the capacity based on the stage
      ProvisionedThroughput:
        ReadCapacityUnits: ${self:custom.tableThroughput}
        WriteCapacityUnits: ${self:custom.tableThroughput}
```

Once these are set up, you are ready for development. You can start off by following the basics [here](https://serverless.com/framework/docs/getting-started/) in the serverless “getting started” section. First, we will add a file called `handler.js` which exports a function `eventQuery` as this is what the template expects to be used as specified in its configuration.

```
functions:
  queryEvents:
    handler: handler.queryEvents
    events:
      - http:
          path: events
          method: post
          cors: true
    environment:
      TABLE_NAME: ${self:custom.tableName}
```

I have made a boilerplate from the things described here with a few enhancements and improvements. I hope this helps you to get started easily. You can go through the `readme.md` on this boilerplate to get started right away. It provides you with serverless yml templates, GraphQL API, Typescript / webpack config, Jest unit testing capabilities, serverless offline, debugging capabilities on VS-Code & CI/CD pipelines on [git-lab](https://about.gitlab.com/) straight away you can find the links below:

> _On the next article i.e Part II , I will be adding a bit more to this to show how I integrated GraphQL._

Check out [Part II here](/going-serverless-with-nodejs-graphql-part-ii-1810445028a4) for the GraphQL implementation.