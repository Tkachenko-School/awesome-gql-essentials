# A new approach to mocking GraphQL data
![A new approach to mocking GraphQL data](https://cdn-media-1.freecodecamp.org/images/1*w8fFdZ3gAy7q1x_qOFB-gg.jpeg)

#### How we power React component tests and examples at Stripe

GraphQL’s main benefit for frontend developers has always been excellent tooling and developer experience. Chief among those is the ability to easily mock your data. API mocking is critical because it lets you write and test your components without having to run your whole app backend. You can even develop UI components based on a mocked schema when the backend implementation isn’t done yet, speeding up development.

In the last few months, the Dashboard Platform team at [Stripe](https://stripe.com/) has been integrating [GraphQL](https://graphql.org/) and [Apollo](https://www.apollographql.com/) for data fetching in the Stripe Dashboard. Our goal is to create a smooth and productive experience for product developers across the whole company. One of the most important aspects of that is making testing as easy as possible. In service of that outcome, we’ve come up with some new patterns that allow us to mock data with an extremely small amount of code.

I’ll tell you how we:

1.  mock GraphQL data for the whole schema
2.  customize our mocks on a per-component basis
3.  mock loading and error states with just one line of code
4.  integrate these mocks into our Jest tests and component explorer

Put together, these new tools allow us to render UI components that depend on GraphQL data in tests and examples, in all of the states that we need them, without writing code to handle specific requests and responses.

So let’s jump right in! We’ve included all of the code needed to follow along in this post. We welcome someone from the community publishing an `npm` package based on our approach.

_Special thanks to my colleagues [Isaac Hellendag](https://twitter.com/hellendag), Oliver Wong, and Jason Divock, who have contributed to these tools and this post._

![](https://cdn-media-1.freecodecamp.org/images/1*DqP8CiEUerOXeYubtxbGRw.png)

How we reduced our component testing boilerplate by eliminating per-request mocks and using a mocked schema.

### Background: Mocking data with graphql-tools

There’s a variety of tools out there that make it super easy to mock requests based on a GraphQL schema and queries.

There’s the original [graphql-tools](https://www.apollographql.com/docs/graphql-tools/mocking.html) library, the [graphql-faker](https://github.com/APIs-guru/graphql-faker) CLI, and now even [Apollo Server has mocking built in](https://www.apollographql.com/docs/apollo-server/features/mocking.html). I’m partial to graphql-tools because it’s the easiest to customize.

Before getting into the new stuff I’m really excited about with per-component customization, I’ll show you the basic mocking setup.

Here’s how you can get a mocked schema up and running super quickly with graphql-tools:
```
import {
  makeExecutableSchema,
  addMockFunctionsToSchema
} from 'graphql-tools';
import { graphql } from 'graphql';

// Fill this in with the schema string
const schemaString = `
  type Todo { id: ID, text: String, completed: Boolean }
  type User { id: ID, name: String }
  type Query { todoItems: [Todo] }
  # ... other types here
`;

// Make a GraphQL schema with no resolvers
const schema = makeExecutableSchema({ typeDefs: schemaString });

// Add mocks, modifies schema in place
addMockFunctionsToSchema({ schema });

const query = `{
  user(id: 6) { id, name }
}`;

graphql(schema, query).then((result) =>
  console.log('Got result', result));

// Result will fill in fake data for all the fields:
// { "data": { "user": {
//   "id": "Hello, world!", "name": "Hello, world"
// }}}
```

This approach lets you generate any shape of fake data, just by providing a query. Here’s how we can wire our mocked schema up to our Apollo-powered components using [apollo-link-schema](https://www.apollographql.com/docs/link/links/schema.html) and Apollo Client:

```
import { ApolloClient } from 'apollo-client';
import { InMemoryCache } from 'apollo-cache-inmemory';
import { SchemaLink } from 'apollo-link-schema';

// Import the schema object from previous code snippet above
import schema from './path/to/your/schema';

const client = new ApolloClient({
  cache: new InMemoryCache(),
  link: new SchemaLink({ schema })
});

// Then, put this client in ApolloProvider
import { ApolloProvider } from 'react-apollo';

// Wherever we want a component to display mocked data
// MyComponent uses the Query component internally
<ApolloProvider client={client}>
  <MyComponent />
</ApolloProvider>
```

Now, we can render a component with mocked data anywhere we want, for example in a Jest test, or a component explorer like Storybook. One nice thing is that `graphql-tools` allows us to pass in custom mocks for our schema on a per-type basis.

```
import faker from 'faker';

const mocks = {
  Todo: () => ({
    text: () => faker.sentence(),
    completed: () => faker.random.boolean(),
  }),
  User: () => ({
    name: () => faker.name.findName()
  })
}

addMockFunctionsToSchema({ schema, mocks });
```

That lets us make sure that the data we get from our mocks looks somewhat real. The `faker` library is super useful here because it lets us get somewhat realistic data with low effort.

Unfortunately, having a mocked schema that returns realistic data isn’t quite enough for a complete mocking setup. Sometimes, you want to have a test or component example display a very specific situation, rather than generic mock data. You also need to make sure your component behaves properly when it gets empty strings, or a really long list, or a loading state or error. And that’s where things get really interesting.

#### Customizing mocks on a per-component basis with a mocking provider

After trying a lot of different approaches, we came up with a neat API that lets us use global mocks while customizing just the types and fields we need to for that particular test or example.

Here’s what it looks like:

```
import TodoList from './TodoList';

// Specify some resolvers for this instance
const customResolvers = {
  Query: () => ({
    todoItems: [
      { completed: true },
      { completed: false },
    ]
  })
};

// TodoList uses the following query:
// query {
//   todoItems {
//     text
//     completed
//     owner { name }
//   }
// }
<ApolloMockingProvider customResolvers={customResolvers}>
  <TodoList />
</ApolloMockingProvider>
```

This allows us to make sure that the component gets exactly two `todo` items, where the first is completed and the second is not. But here’s the best part — the rest of the data comes from the global mocks we have defined for the whole app! **So we only have to specify the fields we care about for this particular example.**

That lets us get the best of both worlds — low effort, realistic global mocks, **while maintaining** the ability to get custom results to demonstrate specific situations on a per-instance basis. So how does it work?

We’ve implemented this via a mocking provider that merges the custom resolvers passed through its props with our global mock resolvers, like this:

```
const ApolloMockingProvider = (props) => {
  const mocks = mergeResolvers(globalMocks, props.customResolvers);
  
  addMockFunctionsToSchema({ schema, mocks });
  
  const client = new ApolloClient({
    link: new SchemaLink({ schema }),
    cache: new InMemoryCache(),
  });
  
  return (
    <ApolloProvider client={client}>
      {props.children}
    </ApolloProvider>
  );
}
```

It takes the custom resolvers you pass in, merges it with your global mocks, and then creates a new Apollo Client instance to be used by the component you are testing.

The most important function here is `mergeResolvers`, which allows us to merge our globally defined mocks which overrides a specific test case. It’s a little too long to fit into this blog post, but it’s about 50 lines of code: [Check out the mergeResolvers function in my coworker Isaac’s Gist.](https://gist.github.com/hellendag/2aa9ad1f9b771f38802760c269bb1b76)

### Mocking loading and error states in one line of code

The system above gets us most of what we need, but it doesn’t have a good way to mock out stuff that’s not actual data — specifically, loading and error states. Thankfully, we can use a similar approach with Apollo Link to create special providers for those cases. For example, here’s a simple provider for mocking a loading state.

```
const LoadingProvider = (props) => {
  const link = new ApolloLink((operation) => {
    return new Observable(() => {});
  });
  
  const client = new ApolloClient({
    link,
    cache: new InMemoryCache(),
  });
  
  return (
    <ApolloProvider client={client}>
      {props.children}
    </ApolloProvider>
  );
};
```

That’s right — it’s so small, it fits in a tweet. And here’s how you would use it:

    <LoadingProvider>
      <TodoList />
    </LoadingProvider>

Super simple! Awesome stuff. And error states are almost as easy.

```
export const ErrorProvider = (props) => {
  // This is just a link that swallows all operations and returns the same thing
  // for every request: The specified error.
  const link = new ApolloLink((operation) => {
    return new Observable((observer) => {
      observer.next({
        errors: props.graphQLErrors || [
          {message: 'Unspecified error from ErrorProvider.'},
        ],
      });
      observer.complete();
    });
  });
  
  const client = new ApolloClient({
    link,
    cache: new InMemoryCache(),
  });
  
  return (
    <ApolloProvider client={client}>
      {props.children}
    </ApolloProvider>
  );
};
```
You can use this in the same way, but you can also pass a customizable error:

    <ErrorProvider graphQLErrors={[{message: 'My error message'}]}>
      <TodoList />
    </ErrorProvider>

Armed with these three tools — the mocked schema provider with custom resolvers, the loading provider, and the error provider — you can achieve common mocking use cases in a very small amount of code.

For the more complex use cases, you can still use the built-in react-apollo [MockedProvider](https://www.apollographql.com/docs/guides/testing-react-components.html#MockedProvider), which lets you specify totally custom request and response pairs.

### Integrating into Jest tests and your component explorer

Now that we’ve got an easy way to mock data, loading states, and errors, we can easily integrate them into Jest or a component explorer. We have our own internal component explorer tool, but a commonly used one in the community is React Storybook.

Here’s what a simple Jest test looks like, using `mount` from [Enzyme](https://github.com/airbnb/enzyme) to render a React component and then check that its contents are what we expect.

```
it('renders a list', async () => {
  const customResolvers = {
    Todo: () => ({
      text: 'My todo item',
    }),
  };
  
  const wrapper = mount(
    <ApolloMockingProvider customResolvers={customResolvers}>
      <TodoList />
    </ApolloMockingProvider>,
  );
  
  await wait(0); // Use a waiting library of your choice
  
  const text = wrapper.text();  
  expect(text).toMatch(/My todo item/);
});
```
And you can use these providers the same way when rendering a component example in Storybook or similar.

```
storiesOf('TodoList', module)
  .add('renders a list', () => {
    const customResolvers = {
      Todo: () => ({
        text: 'My todo item text',
      }),
    };
  
    return (
      <ApolloMockingProvider customResolvers={customResolvers}>
        <TodoList />
      </ApolloMockingProvider>,
    );
  });
```

And that’s how we do it!

### Conclusion

We hope that bringing the power of GraphQL to developers at Stripe will make frontend development much more fun and productive, and this is just the beginning of the story. I’m excited to work with such an awesome team at Stripe!

We’re using our past experience working on frontend teams and technologies to come up with exciting approaches to improve data fetching and API-related tooling. I can’t wait to share more of what we’re working on over the next few months.

Please reach out to me [on Twitter at @stubailo](https://twitter.com/stubailo) if you decide to build a package based on this post, have some feedback, or want to chat about GraphQL and React!

Also, **we’re hiring for many [different engineering roles](https://stripe.com/jobs#engineering) here at Stripe**, so please apply if you want to help us build the economic infrastructure of the internet.