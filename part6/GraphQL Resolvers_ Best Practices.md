# GraphQL Resolvers: Best Practices

![](https://miro.medium.com/max/338/1*fMLqM5yV0HUWBGGYK7beLw.png)

From graphql.org

This post is the first part of a series of best practices and observations we have made while building [GraphQL](https://graphql.org/) APIs at PayPal. In upcoming posts, we’ll share our thoughts on: schema design, error handling, production visibility, optimizing client-side integrations and tooling for teams.

You might have seen our previous post “[GraphQL: A success story for PayPal Checkout”](/paypal-engineering/graphql-a-success-story-for-paypal-checkout-3482f724fb53) about PayPal’s journey from REST to GraphQL. This post dives into details some best practices for building resolvers that are _fast_, _testable_ and _resilient_ over time.

# What’s a resolver?

Let’s start off at the same baseline. What’s a _resolver_?

![](https://miro.medium.com/max/1778/1*ILtky9qYv5SBBijYE0axug.png)
Resolver definition


> Every field on every type is backed by a function called a _resolver_.

A resolver is a function that resolves a value for a type or field in a schema. Resolvers can return objects or scalars like Strings, Numbers, Booleans, etc. If an Object is returned, execution continues to the next child field. If a scalar is returned (typically at a leaf node), execution completes. If null is returned, execution halts and does not continue.

_Resolvers can be asynchronous too!_ They can resolve values from another REST API, database, cache, constant, etc.

Later, we will walk through a series of examples illustrating how to build resolvers that are _fast, testable_ and _resilient._

# Executing queries

To better understand resolvers, you need to know how queries are executed.

Every GraphQL query goes through three phases. Queries are **parsed**, **validated** and **executed**.

1.  **Parse** — A query is **parsed** into an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) (or AST). ASTs are incredibly powerful and behind tools like [ESLint](https://eslint.org/), [babel](https://babeljs.io/), etc. If you want to see what a GraphQL AST looks like, check out [astexplorer.net](https://astexplorer.net/) and change JavaScript to GraphQL. You will see a query on the left and an AST on the right.
2.  **Validate** — The AST is **validated** against the schema. Checks for correct query syntax and if the fields exist.
3.  **Execute** — The runtime **walks** through the AST, starting from the root of the tree, **invokes** resolvers, **collects** up results, and emits JSON.

For this example, we’ll refer to this query:

![](https://miro.medium.com/max/304/1*S3xBNGThKSBNJorDvFzIkQ.png)
Query for later reference

When this query is **parsed**, it’s converted to an AST, or a tree.

![](https://miro.medium.com/max/391/1*hVSDe0UwmZDkwwL13o8Y_A.png)

Query represented as a tree

The root Query type is the entry point to the tree and contains our two root fields, **_user_** and **_album._**The**_user_**and**_album_**resolvers are _executed in parallel_ (which is typical among all runtimes). The tree is executed breadth-first, meaning **_user_** must be resolved before its children **_name_** and **_email_** are executed. If the **_user_** resolver is asynchronous, the **_user_ **branch delays until its resolved. Once all leaf nodes, **_name_, _email_**, **_title_**, are resolved, execution is complete.

Root Query fields, like **_user_** and **_album,_** are executed in parallel but in no particular order. Typically, fields are executed in the order they appear in the query, but it’s not safe to assume that. Because fields are executed in parallel, they are assumed to be _atomic_, _idempotent_, and _side-effect_ free.

# Looking closer at resolvers

In the next few sections, we will use JavaScript, but GraphQL servers can be written in almost any language.

![](https://miro.medium.com/max/876/1*IdRm05pv-cT2rmBY6omFhw.png)

Resolvers with four arguments — **root**, **args**, **context**, **info**


In some form or another, every resolver in every language receives these four arguments:

*   **_root_** — Result from the previous/parent type
*   **_args_** — Arguments provided to the field
*   **_context_** — a _Mutable_ object that is provided to all resolvers
*   <mark class="qf qg mb">**_info_**</mark> <mark class="qf qg mb">— Field-specific information relevant to the query (used rarely)</mark>

These four arguments are core to understanding how data flows between resolvers.

# Default resolvers

Before we continue, it’s worth noting that a GraphQL server has built-in default resolvers, so you don’t have to specify a resolver function for every field. A default resolver will look in **_root_** to find a property with the same name as the field. An implementation likely looks like this:

![](https://miro.medium.com/max/2056/1*x-1OE_wqncx22oJaW3pRkg.png)
Default resolver implementation

# Fetching data in resolvers

Where should we fetch data? What are the tradeoffs

An **event** field has a required **id** argument, returns an **Event**

# Passing data between resolvers

**_context_ **is a mutable Object that is provided to all resolvers. It’s created and destroyed between every request. It is a great place to store common Auth data, common models/fetchers for APIs and databases, etc. At PayPal, we’re a big Node.js shop w/ infrastructure built on [Express](https://expressjs.com/), so we store Express’ **_req_** in there.

When you first learn about **_context_**, an initial thought might be to use **_context_** as a general purpose cache. This is not recommended, but here is what an implementation might look like.

![](https://miro.medium.com/max/1300/1*hFpLRxc1ah53hWZ7B3aRHw.png)

Passing data between resolvers using **context**. This is **not** recommended!

When **_title_** is invoked, we store the event result in **_context_**. When **_photoUrl_** is invoked, we pull event out of **_context_** and use it. This code isn’t reliable. There’s no guarantee that **_title_** will be executed before **_photoUrl_**.

We could fix up both resolvers to check if the event exists in **_context._ **If so, use it. Otherwise, we fetch it and store it for later, but there’s still a _large surface area for mistakes._

Instead, we should _avoid mutating context inside of resolvers_. We should prevent knowledge and concerns from mixing between each other, so that our _resolvers are easy to understand, debug, and test_.

# Passing data from parent-to-child

The** _root_** argument is for passing data from parent resolvers to child resolvers.

For example, if you are building an **Event** type where all fields of Event  
depend on the same data, you might want to fetch it _once_ at the **_event_** field,  
rather than at _every_ field of Event.

Seems like a good idea, right? This is a quick way to get started with building resolvers but you might run into some problems. Let’s understand why.

For the examples below, we’ll work with an **Event** type that has two fields.

![](https://miro.medium.com/max/638/1*ETpAqTtua1lbw1-GJr3PfQ.png)
Event type with two fields: **title** and **photoUrl**

Most of the fields for Event can be fetched from an Event API, so we can fetch it at the top-level **_event_** resolver and provide the results to our **_title_** and **_photoUrl_** resolvers.

![](https://miro.medium.com/max/2092/1*4YmQ1Spyntlt9qMd6hdv3Q.png)

Top-level **event** resolver fetches data, provides results to **title** and **photoUrl** field resolvers

Even better, we don’t need to specify the bottom two resolvers.  
We can use the default resolvers because the Object returned by **_getEvent()_**  
has a **_title_** and **_photoUrl_** property.

![](https://miro.medium.com/max/2092/1*kmfIm7_mlS803lboSbnYkA.png)
**id** and **title** are resolved using default resolvers

_What’s wrong with this?_

There are two scenarios where you might run into overfetching…

## Scenario #1: Multi-layered data fetching

Let’s say some requirements come in and you need to display an event’s attendees. We start by adding an **_attendees_** field to Event.

![](https://miro.medium.com/max/708/1*mH9dZkns250WCXWz6fQiTw.png)

Event type with an additional **attendees** field

When you fetch the attendees details, you have two options: fetch that data at the **_event_** resolver, or **_attendees_** resolver.

We will test out the first option: adding it to the **_event_** resolver.

![](https://miro.medium.com/max/2192/1*HmgJgIUnEtgXBTZdohaq4g.png)
**_event_** resolver calls two APIs, fetching event details and attendees details

If a client queries for only **_title_** and **_photoUrl,_ **but not **_attendees_**.Now you’re being inefficient and making an unnecessary request to your Attendees API.

It’s not your fault, this is how we work. _We recognize patterns and copy them._  
If contributors see that data fetching is done in the **_event_** resolver, they’ll likely  
add any additional data fetching there without thinking too hard about it.

We have one more option to test with fetching the attendees inside of the **_attendees_** resolver.

![](https://miro.medium.com/max/1108/1*iua7h5w6d8nciUKrs6Sk6g.png)

**_attendees_** resolver fetches attendees details from the Attendees API

If our client queries for only **_attendees,_ **not **_title_** and **_photoUrl_**. We’re still being inefficient by making an unnecessary request to our Events API.

## Scenario #2: N+1 Problem

Because data is fetched at a field-level, we run the risk of overfetching. Overfetching and the _N+1 problem_ is a popular topic in the GraphQL world. [Shopify](https://medium.com/u/bab76dfc19b0?source=post_page-----cd36fdbcef55----------------------) has a [great article](https://engineering.shopify.com/blogs/engineering/solving-the-n-1-problem-for-graphql-through-batching) that explains N+1 well.

How does that affect us here?

To illustrate it better, we will add a new **events** field that returns all events.

![](https://miro.medium.com/max/756/1*lL6arOjbuj-bTDnNF0vIUA.png)

An **events** field returns all events.

![](https://miro.medium.com/max/566/1*Z-eRknpzq2uGbT5vnAiPMg.png)

Query for all **events** w/ their **title** and **attendees**

If a client queries for all **events** and their **attendees,** we run the risk of _overfetching_ because **attendees** _can attend more than one event._ We might make duplicate requests for the same attendee. 😬

This problem is amplified in a large organization where requests can fan out and cause unnecessary pressure on your system.

To solve this, we need to batch and de-dupe requests!

In JavaScript, some popular options are [dataloader](https://github.com/facebook/dataloader) and [Apollo data sources](https://www.apollographql.com/docs/apollo-server/features/data-sources.html).

If you’re using another language, there’s likely something you can pick up. So take a look around before solving this on your own.

At the core of it, these libraries sit on top of your data access layer and will cache and de-dupe outgoing requests using debouncing or memoization. If you are curious to what async memoization looks like, check out [Daniel Brain](https://medium.com/u/97f5eca783db?source=post_page-----cd36fdbcef55----------------------)’s [excellent post](/@bluepnume/async-javascript-is-much-more-fun-when-you-spend-less-time-thinking-about-control-flow-8580ce9f73fc)!

# Fetching data at a field-level

Earlier, we saw that it’s easy to get burned by _overfetching_ with “top-heavy” parent-to-child resolvers.

> **Is there a better alternative?**

Let’s tease out the parent-to-child option again. What if we reverse that so that our child _fields_ are responsible for fetching their own data?

![](https://miro.medium.com/max/1300/1*zFxpLudNipF2eDiaIhNBlQ.png)
**Fields** are **responsible** for their **own data fetching**.

> **Why is this a better alternative?**

_This code is easy to reason about._ You know exactly where an email is fetched. This makes for easy debugging.

_This code is more testable._ You don’t have to test the **_event_** resolver when you really just wanted to test the **_title_** resolver.

To some, the getEvent duplication might look like a code smell. _But, having code that is simple, easy to reason about, and is more testable is worth a little bit of duplication._

But, there’s still a potential problem here. If a client queries for **_title_** and **_photoUrl_**, we cause one additional request on our Event API with **_getEvent_**. As we saw earlier in the _N+1 problem_, we should de-dupe requests at a framework level using libraries like [dataloader](https://github.com/facebook/dataloader) and [Apollo data sources](https://www.apollographql.com/docs/apollo-server/features/data-sources.html).

If we _fetch data at a field level_ and _dedupe requests_, we have code that is easier to _debug_ and _test,_ and we can optimally fetch data without thinking about it.

# Best practices

*   Fetching and passing data from parent-to-child should be used sparingly.
*   Use libraries like [dataloader](https://github.com/facebook/dataloader) to de-dupe downstream requests.
*   Be aware of any pressure you’re causing on your data sources.
*   Don’t mutate “context”. Ensures consistent, less buggy code.
*   Write resolvers that are readable, maintainable, testable. Not too clever.
*   Make your resolvers as thin as possible. Extract out data fetching logic to re-usable async functions.

# Stay tuned!

Thoughts? We would love to hear your team’s best practices and learnings with building resolvers. This is a topic that isn’t often discussed but is important to building long-lived GraphQL APIs.

In upcoming posts, we’ll share our thoughts on: schema design, error handling, production visibility, optimizing client-side integrations and tooling for teams.

We’re hiring! 👋 If you would like to come work on front-end infrastructure, GraphQL or React at PayPal, DM me on Twitter at [@mark_stuart](http://twitter.com/mark_stuart)!