# Make web real-time with GraphQL subscriptions

A lot of times we like to provide our users with some real-time features. Consider this simple scenario:

You are making an Instagram web clone.

People start following each other and when one publishes a photo, all people following that user will be notified and will see the photo in their browsers if they have it open. There won’t be any need for a hard reload of the page.

How should we implement that?

Websockets to our help! WebSockets represent a long-awaited evolution in client/server web technology. They allow a long-held single TCP socket connection to be established between the client and server which allows for bi-directional, full duplex, messages to be instantly distributed with little overhead resulting in a very low latency connection.

How could we leverage this in our sample Instagram application?

There are different steps needed before the above scenario could be fulfilled:

1.  The publisher uploads a photo
2.  The server saves the photo in storage
3.  The server will make a list of all people that are following that user
4.  The server will establish a TCP connection with all those users if they are online
5.  The server will send new photo information along with some metadata to those users via that passive TCP connection
6.  Should clients accept WebSockets, they will receive the payload sent to them on the previous step by the server
7.  Clients update the user feed based on the received information

By user feed here, I mean the feed that aggregates the photos from all people that you follow.

The end result will be a soft update for users. They won’t need to do a hard refresh of the browser and they will promptly receive new photos from people that they follow.

Magic!

Let’s get to the implementation. The steps discussed above are very low level. Using the right tools and techniques we could make things more abstract and easier to implement.

There are two components involved in this example:

*   A WebSockets supporting server
*   WebSocket allowing clients


For this example we will be using Rails 4.2.x on the server and React/GraphQL/ModernRelay on the frontend; for the API layer, we will be using graphql-ruby (1.7.9) gem.

The points discussed in this article are not bound to a specific version of the libraries and tools. The same principles are applicable to the newer versions of these tools.

# Background

As mentioned previously, the holy grail of modern web development is a page where every piece updates in real-time and there is no overfetching. In the old days of the Internet, we would send AJAX requests every 10 seconds and we’d get responses back from the server. That would help us to emulate a semi real-time system. The problem with RESTful API has always been overfetching: we want a cup but instead, we get an elephant in a cup.

Subscriptions fix this: instead of polling the server infinitely, the server tells us what’s exactly changed.

Using GraphQL could help us in different ways:

*   It provides support for subscriptions
*   It helps us with the overfetching issue discussed earlier.

![](https://miro.medium.com/max/1758/1*Yt24M2DsefvATAFgTCH_fw.png)


# Subscriptions in GraphQL

_Subscriptions_ allow GraphQL clients to observe specific events and receive updates from the server when those events occur. This supports live updates, such as WebSocket pushes. Subscriptions introduce several new concepts:

*   Subscription Type
*   Triggers
*   Implementation

_SubscriptionType_ is the entrypoint for subscriptions. It complements QueryType and MutationType.

_Triggers_ begin the update process by sending a payload to GraphQL right after an event happens in our application.

The _implementation_ provides application-specific methods for executing & delivering updates.

Our application code must answer the following questions:

*   How does the app keep track of who is subscribed to what?
*   How does the app deliver updates to clients?

# SubscriptionType

For our simple example we are going to add support for one subscription event on the server-side:
```
SubscriptionType = GraphQL::ObjectType.define do  
  name 'Subscription'
  field :feedItemAdded, PhotoType,  
        'A photo is added' do  
    subscription_scope :current_user_uuid  
  end  
end
```
Let’s assume that we have already defined a PhotoType in our GraphQL schema.

Each field mentioned in this SubscriptionType corresponds to an event which may be subscribed to by the clients.

To update certain clientsonly, we can specify a scope for our subscription field. In our example, we are only interested in notifying the people that are following the photo publisher about the new changes; we don’t want to notify all users of our platform. Therefore, we ended up putting a scope on our field.

Scopes are based on query context. Whatever value we have put in context variable for our scope must be passed later to trigger method in order to update the client. To add the **current_user_uuid** field to GQL context, we simply do:
```
ctx = {  
  ...  
  channel_prefix: "private-user-#{current_user.uuid}-",  
  current_user_uuid: current_user.uuid  
}
...  
result = GraphSchema.execute(query, context: ctx, variables: variables)
```
Ignore channel_prefix key value for now; we will come back to it later;

This will guarantee that the context object includes all pieces needed for our subscription scope to work properly.

The last thing that we need to do is to add the subscription root that we just created to our GQL schema.

```
GraphSchema = GraphQL::Schema.define do  
  use SubscriptionType, redis: Redis.new  
  query QueryType  
  mutation MutationType  
  subscription SubscriptionType  
  ...  
end
```

# Triggering events

The next step for the server is to push updates to GraphQL clients with `.trigger`. Events are triggered by their name.

For example, if we want to trigger our feedItemAdded event, we could do it as follows:

```
GraphSchema.subscriptions.trigger(  
  'feedItemAdded',  
  {},  
  new_photo,  
  scope: uuid_of_the_user_that_needs_to_be_notified  
)
```

The arguments are:

*   `name`, which matches the field on the subscription type
*   `arguments`, which corresponds to the arguments on subscription type (in our example we don’t have any)
*   `object`, which will be the root object of the subscription update; here it will be the photo object.
*   `scope:` for implicitly scoping the clients who will receive updates. This is defined based on the value we used earlier for subscription_scope. This guarantees that this update is only received by the client that matches that user_uuid.

We could call this snippet of code right after a photo is created on the server. We will put the logic in the Photo model for now:
```
# photo.rb class photo < ActiveRecord::Base  
   after_create :notify_subscribers_of_addition  
   ...
   def notify_subscribers_of_addition  
     subscribers_uuids.each do |user_uuid|  
       GraphSchema.subscriptions.trigger(  
         'feedItemAdded',  
         {},  
         self,  
         scope: user_uuid  
       )  
     end  
   end
   ...  
end
```
That’s it!

# Implementation

GraphQL::Subscriptions plugin we used earlier is the base class for implementing subscriptions. Each method corresponds to a step in the subscription lifecycle.

graphql-ruby gem supports two different ways of delivering push updates to subscribed clients:

1.  [ActionCable](https://guides.rubyonrails.org/action_cable_overview.html)
2.  [Pusher](https://pusher.com/)

In this article, we will focus on Pusher implementation.

ActionCable became part of Rails starting with version 5\. As a result, a lot of projects that rely on the older version of Rails are incapable of leveraging its power.

The _pusher,_ on the other hand, is supported on a lot of platforms and could be used with Rails 4.2 and lower.

# Pusher Implementation

Pusher implementation relies on Redis.

*   Our application takes GraphQL queries
*   Redis stores subscription data for later updates
*   Pusher sends updates to subscribed clients

According to the docs, the lifecycle of a subscription request is as follows:

*   A `subscription` query is sent to the server
*   The server’s response will include a Pusher channel ID (as an HTTP header) which the client could use to subscribe to
*   The client opens that Pusher channel
*   When the server triggers updates, they’re delivered over the Pusher channel
*   When the client unsubscribes, the server receives a webhook and responds by removing its subscription data

To make this work, we will need a **persistent** Redis database.

# Configuration

Earlier when we defined our GraphSchema, we put the following line there:
```
use SubscriptionType, redis: Redis.new
```
We are basically instructing our SubscriptionType to rely on Redis.

The Redis connection will be used for managing subscription state.

The next thing should be to notify the clients of the pusher channel ID that they need to subscribe to:

```
....  
result = GraphSchema.execute(query, context: ctx, variables: variables)
 if result.try(:subscription?)  
  response.headers['X-Subscription-ID'] = \  
     result.context[:subscription_id]
     ...  
end
render json: result</span></pre>
```
During execution, GraphQL will assign a `subscription_id` to the `context` hash. The client will use that ID to listen for updates, so we must return the `subscription_id` as part of the response headers. This way, the client can use that ID to access the Pusher channel.

Finally, the server needs to receive webhooks from Pusher when clients disconnect. This keeps our local subscription database in sync with Pusher.

In the Pusher web UI, we need to add a webhook for channel existence:

![](https://miro.medium.com/max/3200/1*wQbETK2BnG63bV-jt0G69g.png)

Now let’s add the route for the webhook to our Rails app:

```
Rails.application.routes.draw do  
  ...
  mount GraphSchema.pusher_webhooks_client, at: '/pusher_webhooks'
  ...  
end
```
# Authorization

Pusher Channels are public by default. To ensure the privacy of our subscription updates, we should be using a [private channel](https://pusher.com/docs/client_api_guide/client_private_channels).

The process is fairly simple:

We just need to add a `channel_prefix:` key to our query context which we already did earlier with the following line of code:

```
ctx = {  
  ...  
  channel_prefix: "private-user-#{current_user.uuid}-",  
  ...  
}
```
That prefix will be applied to GraphQL-related Pusher channel names. (According to the Pusher docs, the prefix should begin with `private-` to be considered private)

Now we add a new controller (we call it PusherController):
```
# the following keys are created for you when you create a new pusher project
# you want to put this code in an initializer
PUSHER_CLIENT = Pusher::Client.new(  
  app_id: ENV['PUSHER_APP_ID'],  
  key: ENV['PUSHER_KEY'],  
  secret: ENV['PUSHER_SECRET'],  
  cluster: ENV['PUSHER_CLUSTER']  
)
...
class PusherController < ApplicationController  
  def auth  
    if current_user &&  
       params[:channel_name]  
         .start_with?("private-user-#{current_user.uuid}-")  
      response = PUSHER_CLIENT.authenticate(  
        params[:channel_name],  
        params[:socket_id]  
      )  
      render json: response  
    else  
      render text: 'Forbidden', status: '403'  
    end  
  end  
end
```

Later on, when pusher JS clients try to connect to a channel, they will first hit this endpoint; if the clients are accessing the channels belonging to them, Rails will pass them through and will authenticate them. Afterward, they will be able to interact with the Pusher API.

To expose our controller to outside, we will also need to define a route for it:
```
post '/pusher/auth', to: 'pusher#auth
```
This auth path needs to be passed to our Pusher JS Client when we initialize it later in this guide.

# Client-Side Implementation

To implement the client-side, we will be using [Relay Modern](https://facebook.github.io/relay/en/). Let’s note that we could also use other GraphQL clients like [Apollo](https://www.apollographql.com/).

This article assumes that you are already familiar with basic concepts of Relay Modern.

First, a little set up needs to be done on the client-side.

In order to know how to access your GraphQL server, Relay Modern requires us to provide an object implementing the `NetworkLayer` interface when creating an instance of a [Relay Environment](https://facebook.github.io/relay/docs/en/relay-environment.html). The environment uses this network layer to execute queries, mutations, and subscriptions.

When creating your Relay Modern environment, you want to make sure that a subscription handler is injected into the network:

```
import pusherClient from './pusherClient';  
...
const subscriptionHandler = createHandler({ pusher: pusherClient, fetchOperation });  
// Combine them into a `Network`  
const network = Network.create(fetchQuery, subscriptionHandler);
const environment = new Environment({  
  network,  
  store: new Store(new RecordSource()),  
});
export default environment;</span></pre>
```
In pusherClient.js we will have the following:

```
# pusherClient.js
import Pusher from 'pusher-js';  
import { pusherKey, CSRFToken } from 'hammer/csrf';
const pusherClient = new Pusher(pusherKey, {  
  authEndpoint: '/pusher/auth',  
  auth: {  
    headers: {  
      'Content-Type': 'application/json',  
      'X-XSRF-Token': CSRFToken,  
    },  
  },  
  cluster: 'us2',  
  encrypted: true  
});
export default pusherClient;
```
For _authEndpoint,_ we passed the route we specified earlier. That route is handled by PusherContrller#auth.

All requests issued by pusherClient will be hitting the specified endpoint first and if Rails authorizes the access they will be authenticated. This ensures that no rogue actor is accessing the channels that they are not authorized to.

The last step of the whole process is making the React component that uses this subscription to render elements on the UI.

```
fragment Feed_item on Photo {
    id
    creator {
      id
      name
    }
    photoUrl
}

class Feed extends React.Component {
  componentDidMount() {
    const addItemSubscription = graphql`
      subscription FeedAddSubscription { feedItemAdded { ...Feed_item } }
    `;
    requestSubscription(
      environment, // The relay environment created earlier
      {
        subscription: addItemSubscription,
        variables: null,
        onNext: (response) => {},
        onCompleted: () => {},
        onError: error => console.error(error),
        updater: (store) => {
          // application specific logic goes here
          const item = store.getRootField('feedItemAdded');
          const me = store.getRoot().getLinkedRecord('me');
          const feed = ConnectionHandler.getConnection(me, 'discover_feed');
          const edge = ConnectionHandler.createEdge(
            store,
            feed,
            item,
            'PhotoEdge',
          );
          ConnectionHandler.insertEdgeBefore(feed, edge);
          feed.setValue(feed.getValue('totalCount') + 1, 'totalCount');
        },
      }
    );
  }

  renderContent() {
    ...
  }

  render() {
    return this.renderContent();
  }
}
```

L1-L8: this is the structure of payload that we are expecting from the server for a photo object.

L15: Calling **requestSubscription**, the React client creates a subscription.

L12-L14: **addItemSubscription** variable keeps the subscription field that this component is interested in. We will be passing this variable to requestSubscriptions function discussed previously.

L19: no variable needs to be passed to the server for this subscription field.

L22: **OnError** trigger this function (right now an empty function handler)

L23-L39: we are defining an **updater** function that can supply custom logic for updating the in-memory Relay store based on the server response. Basically what we are trying to say here is that each time a new payload is pushed down to the client by the server, make following changes to the Relay Store. As Relay Store is passively updating, we will see the changes reflected on the UI.
'''
To conclude, using GraphQL subscriptions enable us to create modern real-time web apps. In this article, we stitched together all the required steps in creating a real-time experience for a very basic feature.
Needless to say that the same technique is applicable to mobile apps. For example, if we are using React Native along with GraphQL and Relay, 99% of what we covered here could be applied directly there.
It should be known that in production, things will get more complex. We will need to make sure that our implementation scales well to accommodate it. Scalability techniques weren’t covered in this article for the sake of brevity.
