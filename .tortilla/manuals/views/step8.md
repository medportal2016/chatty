# Step 8: GraphQL Input Types

[//]: # (head-end)


This is the eigth blog in a multipart series building Chatty, a WhatsApp clone, using [React Native](https://facebook.github.io/react-native/) and [Apollo](http://dev.apollodata.com/).

In this tutorial, we’ll focus on adding [GraphQL Input Types](http://graphql.org/learn/schema/#input-types), which will help us clean up our queries and streamline future GraphQL development in our app.

Here’s what we'll accomplish in this tutorial:
1. Discuss writing more flexible GraphQL requests
2. Add GraphQL Input Types to our Schema
3. Update resolvers and business logic to handle Input Types
4. Update client-side GraphQL requests to use Input Types

# Writing flexible GraphQL
So far in our journey, writing GraphQL queries has been a breeze. We've been able to get all the data we need in a single request in exactly the shape we want with very little code. But APIs inevitably get more complex as apps mature. We need to ensure our GraphQL infrastructure adapts gracefully as we expand the functionality of our app.

Let's do an audit of the GraphQL queries that currently power our React Native client to look for opportunities to improve our querying....

You may notice the queries, mutations, and subscriptions that return `Group` types have a similar shape, but don't share any code. If we modify or add a field to the `Group` type later on, we would need to individually update every query and mutation that returns a `Group` type -- not good. 

```
import gql from 'graphql-tag';

import MESSAGE_FRAGMENT from './message.fragment';

// this is our primary group query
const GROUP_QUERY = gql`
  query group($groupId: Int!, $first: Int, $after: String, $last: Int, $before: String) {
    group(id: $groupId) {
      id
      name
      users {
        id
        username
      }
      messages(first: $first, after: $after, last: $last, before: $before) {
        edges {
          cursor
          node {
            ... MessageFragment
          }
        }
        pageInfo {
          hasNextPage
          hasPreviousPage
        }
      }
    }
  }
  ${MESSAGE_FRAGMENT}
`;

// creating a group returns a similar shape for the new group
const CREATE_GROUP_MUTATION = gql`
  mutation createGroup($name: String!, $userIds: [Int!]) {
    createGroup(name: $name, userIds: $userIds) {
      id
      name
      users {
        id
      }
      messages(first: 1) { # we don't need to use variables
        edges {
          cursor
          node {
            ... MessageFragment
          }
        }
      }
    }
  }
  ${MESSAGE_FRAGMENT}
`;

// unsurprisingly subscriptions to new groups
// looks like CREATE_GROUP_MUTATION
const GROUP_ADDED_SUBSCRIPTION = gql`
  subscription onGroupAdded($userId: Int){
    groupAdded(userId: $userId){
      id
      name
      messages(first: 1) {
        edges {
          cursor
          node {
            ... MessageFragment
          }
        }
      }
    }
  }
  ${MESSAGE_FRAGMENT}
`;

// and the group field in USER_QUERY looks a lot like these too
export const USER_QUERY = gql`
  query user($id: Int) {
    user(id: $id) {
      id
      email
      username
      groups {
        id
        name
        messages(first: 1) { # we don't need to use variables
          edges {
            cursor
            node {
              ... MessageFragment
            }
          }
        }
      }
      friends {
        id
        username
      }
    }
  }
  ${MESSAGE_FRAGMENT}
`;
```

If we create a common GraphQL fragment for our queries and mutations to share, we'll only need to update the one fragment when the `Group` type changes and all our queries, mutations, and subscriptions will benefit:

[{]: <helper> (diffStep 8.1)

#### [Step 8.1: Create GROUP_FRAGMENT](https://github.com/srtucker22/chatty/commit/aeee1cf)

##### Added client&#x2F;src&#x2F;graphql&#x2F;group.fragment.js
```diff
@@ -0,0 +1,29 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊
+┊  ┊ 3┊import MESSAGE_FRAGMENT from './message.fragment';
+┊  ┊ 4┊
+┊  ┊ 5┊const GROUP_FRAGMENT = gql`
+┊  ┊ 6┊  fragment GroupFragment on Group {
+┊  ┊ 7┊    id
+┊  ┊ 8┊    name
+┊  ┊ 9┊    users {
+┊  ┊10┊      id
+┊  ┊11┊      username
+┊  ┊12┊    }
+┊  ┊13┊    messages(first: $first, last: $last, before: $before, after: $after) {
+┊  ┊14┊      edges {
+┊  ┊15┊        cursor
+┊  ┊16┊        node {
+┊  ┊17┊          ... MessageFragment
+┊  ┊18┊        }
+┊  ┊19┊      }
+┊  ┊20┊      pageInfo {
+┊  ┊21┊        hasNextPage
+┊  ┊22┊        hasPreviousPage
+┊  ┊23┊      }
+┊  ┊24┊    }
+┊  ┊25┊  }
+┊  ┊26┊  ${MESSAGE_FRAGMENT}
+┊  ┊27┊`;
+┊  ┊28┊
+┊  ┊29┊export default GROUP_FRAGMENT;
```

[}]: #

Now we can update all these GraphQL requests to use the fragment:

[{]: <helper> (diffStep 8.2)

#### [Step 8.2: Apply GROUP_FRAGMENT to Queries with default variables](https://github.com/srtucker22/chatty/commit/a3f1cc1)

##### Changed client&#x2F;src&#x2F;graphql&#x2F;create-group.mutation.js
```diff
@@ -1,26 +1,14 @@
 ┊ 1┊ 1┊import gql from 'graphql-tag';
 ┊ 2┊ 2┊
-┊ 3┊  ┊import MESSAGE_FRAGMENT from './message.fragment';
+┊  ┊ 3┊import GROUP_FRAGMENT from './group.fragment';
 ┊ 4┊ 4┊
 ┊ 5┊ 5┊const CREATE_GROUP_MUTATION = gql`
-┊ 6┊  ┊  mutation createGroup($name: String!, $userIds: [Int!]) {
+┊  ┊ 6┊  mutation createGroup($name: String!, $userIds: [Int!], $first: Int = 1, $after: String, $last: Int, $before: String) {
 ┊ 7┊ 7┊    createGroup(name: $name, userIds: $userIds) {
-┊ 8┊  ┊      id
-┊ 9┊  ┊      name
-┊10┊  ┊      users {
-┊11┊  ┊        id
-┊12┊  ┊      }
-┊13┊  ┊      messages(first: 1) { # we don't need to use variables
-┊14┊  ┊        edges {
-┊15┊  ┊          cursor
-┊16┊  ┊          node {
-┊17┊  ┊            ... MessageFragment
-┊18┊  ┊          }
-┊19┊  ┊        }
-┊20┊  ┊      }
+┊  ┊ 8┊      ... GroupFragment
 ┊21┊ 9┊    }
 ┊22┊10┊  }
-┊23┊  ┊  ${MESSAGE_FRAGMENT}
+┊  ┊11┊  ${GROUP_FRAGMENT}
 ┊24┊12┊`;
 ┊25┊13┊
 ┊26┊14┊export default CREATE_GROUP_MUTATION;
```

##### Changed client&#x2F;src&#x2F;graphql&#x2F;group-added.subscription.js
```diff
@@ -1,23 +1,14 @@
 ┊ 1┊ 1┊import gql from 'graphql-tag';
 ┊ 2┊ 2┊
-┊ 3┊  ┊import MESSAGE_FRAGMENT from './message.fragment';
+┊  ┊ 3┊import GROUP_FRAGMENT from './group.fragment';
 ┊ 4┊ 4┊
 ┊ 5┊ 5┊const GROUP_ADDED_SUBSCRIPTION = gql`
-┊ 6┊  ┊  subscription onGroupAdded($userId: Int){
+┊  ┊ 6┊  subscription onGroupAdded($userId: Int, $first: Int = 1, $after: String, $last: Int, $before: String){
 ┊ 7┊ 7┊    groupAdded(userId: $userId){
-┊ 8┊  ┊      id
-┊ 9┊  ┊      name
-┊10┊  ┊      messages(first: 1) {
-┊11┊  ┊        edges {
-┊12┊  ┊          cursor
-┊13┊  ┊          node {
-┊14┊  ┊            ... MessageFragment
-┊15┊  ┊          }
-┊16┊  ┊        }
-┊17┊  ┊      }
+┊  ┊ 8┊      ... GroupFragment
 ┊18┊ 9┊    }
 ┊19┊10┊  }
-┊20┊  ┊  ${MESSAGE_FRAGMENT}
+┊  ┊11┊  ${GROUP_FRAGMENT}
 ┊21┊12┊`;
 ┊22┊13┊
 ┊23┊14┊export default GROUP_ADDED_SUBSCRIPTION;
```

##### Changed client&#x2F;src&#x2F;graphql&#x2F;group.query.js
```diff
@@ -1,31 +1,14 @@
 ┊ 1┊ 1┊import gql from 'graphql-tag';
 ┊ 2┊ 2┊
-┊ 3┊  ┊import MESSAGE_FRAGMENT from './message.fragment';
+┊  ┊ 3┊import GROUP_FRAGMENT from './group.fragment';
 ┊ 4┊ 4┊
 ┊ 5┊ 5┊const GROUP_QUERY = gql`
-┊ 6┊  ┊  query group($groupId: Int!, $first: Int, $after: String, $last: Int, $before: String) {
+┊  ┊ 6┊  query group($groupId: Int!, $first: Int = 1, $after: String, $last: Int, $before: String) {
 ┊ 7┊ 7┊    group(id: $groupId) {
-┊ 8┊  ┊      id
-┊ 9┊  ┊      name
-┊10┊  ┊      users {
-┊11┊  ┊        id
-┊12┊  ┊        username
-┊13┊  ┊      }
-┊14┊  ┊      messages(first: $first, after: $after, last: $last, before: $before) {
-┊15┊  ┊        edges {
-┊16┊  ┊          cursor
-┊17┊  ┊          node {
-┊18┊  ┊            ... MessageFragment
-┊19┊  ┊          }
-┊20┊  ┊        }
-┊21┊  ┊        pageInfo {
-┊22┊  ┊          hasNextPage
-┊23┊  ┊          hasPreviousPage
-┊24┊  ┊        }
-┊25┊  ┊      }
+┊  ┊ 8┊      ... GroupFragment
 ┊26┊ 9┊    }
 ┊27┊10┊  }
-┊28┊  ┊  ${MESSAGE_FRAGMENT}
+┊  ┊11┊  ${GROUP_FRAGMENT}
 ┊29┊12┊`;
 ┊30┊13┊
 ┊31┊14┊export default GROUP_QUERY;
```

[}]: #

There are a few things worth noting about this pattern:
1. Changing fields on `GROUP_FRAGMENT` will immediately apply to all queries, mutations, and subscriptions that use it.
2. We are occasionally using default values for the `$first` variable -- `$first: Int = 1` to return the first message in a Group if that variable is not specified when executing the query/mutation/subscription. 

(GraphQL default variables is without a doubt the greatest and most essential addition to `apollo-client` of all time, and whoever wrote that PR deserves free beer for life 😉)

3. Our GraphQL requests have much simpler return shapes, but much more complex sets of variables.

Old `CREATE_GROUP_MUTATION`:
```
mutation createGroup($name: String!, $userIds: [Int!]) { ... }
```

New `CREATE_GROUP_MUTATION`:
```
mutation createGroup($name: String!, $userIds: [Int!], $first: Int = 1, $after: String, $last: Int, $before: String) { ... }
```

Yeesh! If we needed to change a variable used in `GROUP_FRAGMENT`, we'd still have to change all the queries/mutations/subscriptions. Moreover, it's not very clear what all these variables mean. `$first`, `$after`, `$last`, and `$before` are variables we use to paginate messages within a `Group`, but those variables need to be specified in `USER_QUERY` -- that's nonobvious and weird. What we need is a way to abstract inputs to simplify the way we declare variables and update those variables as our app evolves. Enter GraphQL Input Types!

# Input Types on the Server
GraphQL Input Types are a super simple concept -- you can declare named arguments in a GraphQL request in whatever shape you want. 

For example, we can abstract away the pagination variables from our GraphQL requests by adding the following `ConnectionInput` in our schema:

```
# input for relay cursor connections
  input ConnectionInput {
    first: Int
    after: String
    last: Int
    before: String
  }
```

This will enable us to update `Group` like so:

```
# a group chat entity
  type Group {
    id: Int! # unique id for the group
    name: String # name of the group
    users: [User]! # users in the group
    messages(messageConnection: ConnectionInput): MessageConnection # messages sent to the group
  }
```

This will drastically simplify any request that returns `Group` types!

We should strive to apply input types to all of our GraphQL requests that have even the slightest complexity in their input requirements. For Chatty, I've added input types for most of our mutations:

[{]: <helper> (diffStep 8.3)

#### [Step 8.3: Add Input Types to Schema](https://github.com/srtucker22/chatty/commit/a001d35)

##### Changed server&#x2F;data&#x2F;schema.js
```diff
@@ -4,6 +4,45 @@
 ┊ 4┊ 4┊  # declare custom scalars
 ┊ 5┊ 5┊  scalar Date
 ┊ 6┊ 6┊
+┊  ┊ 7┊  # input for creating messages
+┊  ┊ 8┊  input CreateMessageInput {
+┊  ┊ 9┊    groupId: Int!
+┊  ┊10┊    text: String!
+┊  ┊11┊  }
+┊  ┊12┊
+┊  ┊13┊  # input for creating groups
+┊  ┊14┊  input CreateGroupInput {
+┊  ┊15┊    name: String!
+┊  ┊16┊    userIds: [Int!]
+┊  ┊17┊  }
+┊  ┊18┊
+┊  ┊19┊  # input for updating groups
+┊  ┊20┊  input UpdateGroupInput {
+┊  ┊21┊    id: Int!
+┊  ┊22┊    name: String
+┊  ┊23┊    userIds: [Int!]
+┊  ┊24┊  }
+┊  ┊25┊
+┊  ┊26┊  # input for signing in users
+┊  ┊27┊  input SigninUserInput {
+┊  ┊28┊    email: String!
+┊  ┊29┊    password: String!
+┊  ┊30┊    username: String
+┊  ┊31┊  }
+┊  ┊32┊
+┊  ┊33┊  # input for updating users
+┊  ┊34┊  input UpdateUserInput {
+┊  ┊35┊    username: String
+┊  ┊36┊  }
+┊  ┊37┊
+┊  ┊38┊  # input for relay cursor connections
+┊  ┊39┊  input ConnectionInput {
+┊  ┊40┊    first: Int
+┊  ┊41┊    after: String
+┊  ┊42┊    last: Int
+┊  ┊43┊    before: String
+┊  ┊44┊  }
+┊  ┊45┊
 ┊ 7┊46┊  type MessageConnection {
 ┊ 8┊47┊    edges: [MessageEdge]
 ┊ 9┊48┊    pageInfo: PageInfo!
```
```diff
@@ -24,7 +63,7 @@
 ┊24┊63┊    id: Int! # unique id for the group
 ┊25┊64┊    name: String # name of the group
 ┊26┊65┊    users: [User]! # users in the group
-┊27┊  ┊    messages(first: Int, after: String, last: Int, before: String): MessageConnection # messages sent to the group
+┊  ┊66┊    messages(messageConnection: ConnectionInput): MessageConnection # messages sent to the group
 ┊28┊67┊  }
 ┊29┊68┊
 ┊30┊69┊  # a user -- keep type really simple for now
```
```diff
@@ -62,13 +101,13 @@
 ┊ 62┊101┊
 ┊ 63┊102┊  type Mutation {
 ┊ 64┊103┊    # send a message to a group
-┊ 65┊   ┊    createMessage(text: String!, groupId: Int!): Message
-┊ 66┊   ┊    createGroup(name: String!, userIds: [Int]): Group
+┊   ┊104┊    createMessage(message: CreateMessageInput!): Message
+┊   ┊105┊    createGroup(group: CreateGroupInput!): Group
 ┊ 67┊106┊    deleteGroup(id: Int!): Group
 ┊ 68┊107┊    leaveGroup(id: Int!): Group # let user leave group
-┊ 69┊   ┊    updateGroup(id: Int!, name: String): Group
-┊ 70┊   ┊    login(email: String!, password: String!): User
-┊ 71┊   ┊    signup(email: String!, password: String!, username: String): User
+┊   ┊108┊    updateGroup(group: UpdateGroupInput!): Group
+┊   ┊109┊    login(user: SigninUserInput!): User
+┊   ┊110┊    signup(user: SigninUserInput!): User
 ┊ 72┊111┊  }
 ┊ 73┊112┊
 ┊ 74┊113┊  type Subscription {
```

[}]: #

Sweet! Now let's update our resolvers and business logic to handle input types instead of individual variables. The changes are minimal:

[{]: <helper> (diffStep 8.4)

#### [Step 8.4: Add Input Types to Resolvers and Logic](https://github.com/srtucker22/chatty/commit/e5b4c86)

##### Changed server&#x2F;data&#x2F;logic.js
```diff
@@ -18,7 +18,9 @@
 ┊18┊18┊  to(message) {
 ┊19┊19┊    return message.getGroup({ attributes: ['id', 'name'] });
 ┊20┊20┊  },
-┊21┊  ┊  createMessage(_, { text, groupId }, ctx) {
+┊  ┊21┊  createMessage(_, createMessageInput, ctx) {
+┊  ┊22┊    const { text, groupId } = createMessageInput.message;
+┊  ┊23┊
 ┊22┊24┊    return getAuthenticatedUser(ctx)
 ┊23┊25┊      .then(user => user.getGroups({ where: { id: groupId }, attributes: ['id'] })
 ┊24┊26┊        .then((group) => {
```
```diff
@@ -38,7 +40,9 @@
 ┊38┊40┊  users(group) {
 ┊39┊41┊    return group.getUsers({ attributes: ['id', 'username'] });
 ┊40┊42┊  },
-┊41┊  ┊  messages(group, { first, last, before, after }) {
+┊  ┊43┊  messages(group, { messageConnection = {} }) {
+┊  ┊44┊    const { first, last, before, after } = messageConnection;
+┊  ┊45┊
 ┊42┊46┊    // base query -- get messages from the right group
 ┊43┊47┊    const where = { groupId: group.id };
 ┊44┊48┊
```
```diff
@@ -105,7 +109,9 @@
 ┊105┊109┊      }],
 ┊106┊110┊    }));
 ┊107┊111┊  },
-┊108┊   ┊  createGroup(_, { name, userIds }, ctx) {
+┊   ┊112┊  createGroup(_, createGroupInput, ctx) {
+┊   ┊113┊    const { name, userIds } = createGroupInput.group;
+┊   ┊114┊
 ┊109┊115┊    return getAuthenticatedUser(ctx)
 ┊110┊116┊      .then(user => user.getFriends({ where: { id: { $in: userIds } } })
 ┊111┊117┊        .then((friends) => { // eslint-disable-line arrow-body-style
```
```diff
@@ -158,8 +164,10 @@
 ┊158┊164┊      });
 ┊159┊165┊    });
 ┊160┊166┊  },
-┊161┊   ┊  updateGroup(_, { id, name }, ctx) {
-┊162┊   ┊    return getAuthenticatedUser(ctx).then((user) => { // eslint-disable-line arrow-body-style
+┊   ┊167┊  updateGroup(_, updateGroupInput, ctx) {
+┊   ┊168┊    const { id, name } = updateGroupInput.group;
+┊   ┊169┊
+┊   ┊170┊    return getAuthenticatedUser(ctx).then((user) => {  // eslint-disable-line arrow-body-style
 ┊163┊171┊      return Group.findOne({
 ┊164┊172┊        where: { id },
 ┊165┊173┊        include: [{
```

##### Changed server&#x2F;data&#x2F;resolvers.js
```diff
@@ -55,8 +55,10 @@
 ┊55┊55┊    updateGroup(_, args, ctx) {
 ┊56┊56┊      return groupLogic.updateGroup(_, args, ctx);
 ┊57┊57┊    },
-┊58┊  ┊    login(_, { email, password }, ctx) {
+┊  ┊58┊    login(_, signinUserInput, ctx) {
 ┊59┊59┊      // find user by email
+┊  ┊60┊      const { email, password } = signinUserInput.user;
+┊  ┊61┊
 ┊60┊62┊      return User.findOne({ where: { email } }).then((user) => {
 ┊61┊63┊        if (user) {
 ┊62┊64┊          // validate password
```
```diff
@@ -80,7 +82,9 @@
 ┊80┊82┊        return Promise.reject('email not found');
 ┊81┊83┊      });
 ┊82┊84┊    },
-┊83┊  ┊    signup(_, { email, password, username }, ctx) {
+┊  ┊85┊    signup(_, signinUserInput, ctx) {
+┊  ┊86┊      const { email, password, username } = signinUserInput.user;
+┊  ┊87┊
 ┊84┊88┊      // find user by email
 ┊85┊89┊      return User.findOne({ where: { email } }).then((existing) => {
 ┊86┊90┊        if (!existing) {
```

[}]: #

That's it!

# Input Types on the Client
We need the GraphQL requests on our client to match the input type updates we made on our server.

Let's start by updating `GROUP_FRAGMENT` with our new `ConnectionInput`:

[{]: <helper> (diffStep 8.5 files="client/src/graphql/group.fragment.js")

#### [Step 8.5: Add Input Types to Mutations](https://github.com/srtucker22/chatty/commit/74c690d)

##### Changed client&#x2F;src&#x2F;graphql&#x2F;group.fragment.js
```diff
@@ -10,7 +10,7 @@
 ┊10┊10┊      id
 ┊11┊11┊      username
 ┊12┊12┊    }
-┊13┊  ┊    messages(first: $first, last: $last, before: $before, after: $after) {
+┊  ┊13┊    messages(messageConnection: $messageConnection) {
 ┊14┊14┊      edges {
 ┊15┊15┊        cursor
 ┊16┊16┊        node {
```

[}]: #

This will super simplify all GraphQL requests that return `Group` types:

[{]: <helper> (diffStep 8.5 files="client/src/graphql/group.query.js,client/src/graphql/create-group.mutation.js,client/src/graphql/group-added.subscription.js")

#### [Step 8.5: Add Input Types to Mutations](https://github.com/srtucker22/chatty/commit/74c690d)

##### Changed client&#x2F;src&#x2F;graphql&#x2F;create-group.mutation.js
```diff
@@ -3,8 +3,8 @@
 ┊ 3┊ 3┊import GROUP_FRAGMENT from './group.fragment';
 ┊ 4┊ 4┊
 ┊ 5┊ 5┊const CREATE_GROUP_MUTATION = gql`
-┊ 6┊  ┊  mutation createGroup($name: String!, $userIds: [Int!], $first: Int = 1, $after: String, $last: Int, $before: String) {
-┊ 7┊  ┊    createGroup(name: $name, userIds: $userIds) {
+┊  ┊ 6┊  mutation createGroup($group: CreateGroupInput!, $messageConnection: ConnectionInput = { first: 1 }) {
+┊  ┊ 7┊    createGroup(group: $group) {
 ┊ 8┊ 8┊      ... GroupFragment
 ┊ 9┊ 9┊    }
 ┊10┊10┊  }
```

##### Changed client&#x2F;src&#x2F;graphql&#x2F;group-added.subscription.js
```diff
@@ -3,7 +3,7 @@
 ┊3┊3┊import GROUP_FRAGMENT from './group.fragment';
 ┊4┊4┊
 ┊5┊5┊const GROUP_ADDED_SUBSCRIPTION = gql`
-┊6┊ ┊  subscription onGroupAdded($userId: Int, $first: Int = 1, $after: String, $last: Int, $before: String){
+┊ ┊6┊  subscription onGroupAdded($userId: Int, $messageConnection: ConnectionInput){
 ┊7┊7┊    groupAdded(userId: $userId){
 ┊8┊8┊      ... GroupFragment
 ┊9┊9┊    }
```

##### Changed client&#x2F;src&#x2F;graphql&#x2F;group.query.js
```diff
@@ -3,7 +3,7 @@
 ┊3┊3┊import GROUP_FRAGMENT from './group.fragment';
 ┊4┊4┊
 ┊5┊5┊const GROUP_QUERY = gql`
-┊6┊ ┊  query group($groupId: Int!, $first: Int = 1, $after: String, $last: Int, $before: String) {
+┊ ┊6┊  query group($groupId: Int!, $messageConnection: ConnectionInput = {first: 0}) {
 ┊7┊7┊    group(id: $groupId) {
 ┊8┊8┊      ... GroupFragment
 ┊9┊9┊    }
```

[}]: #

Our other mutations will also look cleaner with their fancy input types as well:

[{]: <helper> (diffStep 8.5 files="client/src/graphql/login.mutation.js,client/src/graphql/signup.mutation.js,client/src/graphql/create-message.mutation.js")

#### [Step 8.5: Add Input Types to Mutations](https://github.com/srtucker22/chatty/commit/74c690d)

##### Changed client&#x2F;src&#x2F;graphql&#x2F;create-message.mutation.js
```diff
@@ -3,8 +3,8 @@
 ┊ 3┊ 3┊import MESSAGE_FRAGMENT from './message.fragment';
 ┊ 4┊ 4┊
 ┊ 5┊ 5┊const CREATE_MESSAGE_MUTATION = gql`
-┊ 6┊  ┊  mutation createMessage($text: String!, $groupId: Int!) {
-┊ 7┊  ┊    createMessage(text: $text, groupId: $groupId) {
+┊  ┊ 6┊  mutation createMessage($message: CreateMessageInput!) {
+┊  ┊ 7┊    createMessage(message: $message) {
 ┊ 8┊ 8┊      ... MessageFragment
 ┊ 9┊ 9┊    }
 ┊10┊10┊  }
```

##### Changed client&#x2F;src&#x2F;graphql&#x2F;login.mutation.js
```diff
@@ -1,8 +1,8 @@
 ┊1┊1┊import gql from 'graphql-tag';
 ┊2┊2┊
 ┊3┊3┊const LOGIN_MUTATION = gql`
-┊4┊ ┊  mutation login($email: String!, $password: String!) {
-┊5┊ ┊    login(email: $email, password: $password) {
+┊ ┊4┊  mutation login($user: SigninUserInput!) {
+┊ ┊5┊    login(user: $user) {
 ┊6┊6┊      id
 ┊7┊7┊      jwt
 ┊8┊8┊      username
```

##### Changed client&#x2F;src&#x2F;graphql&#x2F;signup.mutation.js
```diff
@@ -1,8 +1,8 @@
 ┊1┊1┊import gql from 'graphql-tag';
 ┊2┊2┊
 ┊3┊3┊const SIGNUP_MUTATION = gql`
-┊4┊ ┊  mutation signup($email: String!, $password: String!) {
-┊5┊ ┊    signup(email: $email, password: $password) {
+┊ ┊4┊  mutation signup($user: SigninUserInput!) {
+┊ ┊5┊    signup(user: $user) {
 ┊6┊6┊      id
 ┊7┊7┊      jwt
 ┊8┊8┊      username
```

[}]: #

Finally, we need to update our React Native components to pass in the right values to the new input types. The changes are pretty trivial:

[{]: <helper> (diffStep 8.6)

#### [Step 8.6: Add Input Types to Screens](https://github.com/srtucker22/chatty/commit/eb6074b)

##### Changed client&#x2F;src&#x2F;screens&#x2F;finalize-group.screen.js
```diff
@@ -223,9 +223,9 @@
 ┊223┊223┊
 ┊224┊224┊const createGroupMutation = graphql(CREATE_GROUP_MUTATION, {
 ┊225┊225┊  props: ({ ownProps, mutate }) => ({
-┊226┊   ┊    createGroup: ({ name, userIds }) =>
+┊   ┊226┊    createGroup: group =>
 ┊227┊227┊      mutate({
-┊228┊   ┊        variables: { name, userIds },
+┊   ┊228┊        variables: { group },
 ┊229┊229┊        update: (store, { data: { createGroup } }) => {
 ┊230┊230┊          // Read the data from our cache for this query.
 ┊231┊231┊          const data = store.readQuery({ query: USER_QUERY, variables: { id: ownProps.auth.id } });
```

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
```diff
@@ -264,7 +264,9 @@
 ┊264┊264┊  options: ownProps => ({
 ┊265┊265┊    variables: {
 ┊266┊266┊      groupId: ownProps.navigation.state.params.groupId,
-┊267┊   ┊      first: ITEMS_PER_PAGE,
+┊   ┊267┊      messageConnection: {
+┊   ┊268┊        first: ITEMS_PER_PAGE,
+┊   ┊269┊      },
 ┊268┊270┊    },
 ┊269┊271┊  }),
 ┊270┊272┊  props: ({ data: { fetchMore, loading, group, refetch, subscribeToMore } }) => ({
```
```diff
@@ -278,7 +280,10 @@
 ┊278┊280┊        // GROUP_QUERY is used by default)
 ┊279┊281┊        variables: {
 ┊280┊282┊          // load more queries starting from the cursor of the last (oldest) message
-┊281┊   ┊          after: group.messages.edges[group.messages.edges.length - 1].cursor,
+┊   ┊283┊          messageConnection: {
+┊   ┊284┊            first: ITEMS_PER_PAGE,
+┊   ┊285┊            after: group.messages.edges[group.messages.edges.length - 1].cursor,
+┊   ┊286┊          },
 ┊282┊287┊        },
 ┊283┊288┊        updateQuery: (previousResult, { fetchMoreResult }) => {
 ┊284┊289┊          // we will make an extra call to check if no more entries
```
```diff
@@ -300,15 +305,15 @@
 ┊300┊305┊
 ┊301┊306┊const createMessageMutation = graphql(CREATE_MESSAGE_MUTATION, {
 ┊302┊307┊  props: ({ ownProps, mutate }) => ({
-┊303┊   ┊    createMessage: ({ text, groupId }) =>
+┊   ┊308┊    createMessage: message =>
 ┊304┊309┊      mutate({
-┊305┊   ┊        variables: { text, groupId },
+┊   ┊310┊        variables: { message },
 ┊306┊311┊        optimisticResponse: {
 ┊307┊312┊          __typename: 'Mutation',
 ┊308┊313┊          createMessage: {
 ┊309┊314┊            __typename: 'Message',
 ┊310┊315┊            id: -1, // don't know id yet, but it doesn't matter
-┊311┊   ┊            text, // we know what the text will be
+┊   ┊316┊            text: message.text, // we know what the text will be
 ┊312┊317┊            createdAt: new Date().toISOString(), // the time is now!
 ┊313┊318┊            from: {
 ┊314┊319┊              __typename: 'User',
```
```diff
@@ -317,7 +322,7 @@
 ┊317┊322┊            },
 ┊318┊323┊            to: {
 ┊319┊324┊              __typename: 'Group',
-┊320┊   ┊              id: groupId,
+┊   ┊325┊              id: message.groupId,
 ┊321┊326┊            },
 ┊322┊327┊          },
 ┊323┊328┊        },
```
```diff
@@ -326,8 +331,8 @@
 ┊326┊331┊          const groupData = store.readQuery({
 ┊327┊332┊            query: GROUP_QUERY,
 ┊328┊333┊            variables: {
-┊329┊   ┊              groupId,
-┊330┊   ┊              first: ITEMS_PER_PAGE,
+┊   ┊334┊              groupId: message.groupId,
+┊   ┊335┊              messageConnection: { first: ITEMS_PER_PAGE },
 ┊331┊336┊            },
 ┊332┊337┊          });
 ┊333┊338┊
```
```diff
@@ -342,8 +347,8 @@
 ┊342┊347┊          store.writeQuery({
 ┊343┊348┊            query: GROUP_QUERY,
 ┊344┊349┊            variables: {
-┊345┊   ┊              groupId,
-┊346┊   ┊              first: ITEMS_PER_PAGE,
+┊   ┊350┊              groupId: message.groupId,
+┊   ┊351┊              messageConnection: { first: ITEMS_PER_PAGE },
 ┊347┊352┊            },
 ┊348┊353┊            data: groupData,
 ┊349┊354┊          });
```
```diff
@@ -356,9 +361,9 @@
 ┊356┊361┊          });
 ┊357┊362┊
 ┊358┊363┊          // check whether the mutation is the latest message and update cache
-┊359┊   ┊          const updatedGroup = _.find(userData.user.groups, { id: groupId });
+┊   ┊364┊          const updatedGroup = _.find(userData.user.groups, { id: message.groupId });
 ┊360┊365┊          if (!updatedGroup.messages.edges.length ||
-┊361┊   ┊            moment(updatedGroup.messages.edges[0].node.createdAt).isBefore(moment(createMessage.createdAt))) {
+┊   ┊366┊            moment(updatedGroup.messages.edges[0].node.createdAt).isBefore(moment(message.createdAt))) {
 ┊362┊367┊            // update the latest message
 ┊363┊368┊            updatedGroup.messages.edges[0] = {
 ┊364┊369┊              __typename: 'MessageEdge',
```

##### Changed client&#x2F;src&#x2F;screens&#x2F;signin.screen.js
```diff
@@ -212,18 +212,18 @@
 ┊212┊212┊
 ┊213┊213┊const login = graphql(LOGIN_MUTATION, {
 ┊214┊214┊  props: ({ mutate }) => ({
-┊215┊   ┊    login: ({ email, password }) =>
+┊   ┊215┊    login: user =>
 ┊216┊216┊      mutate({
-┊217┊   ┊        variables: { email, password },
+┊   ┊217┊        variables: { user },
 ┊218┊218┊      }),
 ┊219┊219┊  }),
 ┊220┊220┊});
 ┊221┊221┊
 ┊222┊222┊const signup = graphql(SIGNUP_MUTATION, {
 ┊223┊223┊  props: ({ mutate }) => ({
-┊224┊   ┊    signup: ({ email, password }) =>
+┊   ┊224┊    signup: user =>
 ┊225┊225┊      mutate({
-┊226┊   ┊        variables: { email, password },
+┊   ┊226┊        variables: { user },
 ┊227┊227┊      }),
 ┊228┊228┊  }),
 ┊229┊229┊});
```

[}]: #

Unlike with the previous tutorials in this series, this one doesn't have a flashy ending. Everything should be working as if nothing ever happenend, but under the hood, we've vastly improved the way we make GraphQL requests to gracefully adapt to future changes to our Schema! 

Fragments, default variables, and input types are essential tools for designing scalable GraphQL schemas to use in everchanging complex applications. They keep our code lean and adaptable. Apply liberally!

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/srtucker22/chatty/tree/master@2.0.0/.tortilla/manuals/views/step7.md) |
|:----------------------|

[}]: #
