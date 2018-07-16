# Step 4: GraphQL Mutations

[//]: # (head-end)


This is the fourth blog in a multipart series where we will be building Chatty, a WhatsApp clone, using [React Native](https://facebook.github.io/react-native/) and [Apollo](http://dev.apollodata.com/).

Here’s what we will accomplish in this tutorial:
1. Design **GraphQL Mutations** and add them to the GraphQL Schemas on our server
2. Modify the layout on our React Native client to let users send Messages
3. Build GraphQL Mutations on our RN client and connect them to components using `react-apollo`
4. Add **Optimistic UI** to our GraphQL Mutations so our RN client updates as soon as the Message is sent — even before the server sends a response!

***YOUR CHALLENGE***
1. Add GraphQL Mutations on our server for creating, modifying, and deleting Groups
2. Add new Screens to our React Native app for creating, modifying, and deleting Groups
3. Build GraphQL Queries and Mutations for our new Screens and connect them using `react-apollo`

# Adding GraphQL Mutations on the Server
While GraphQL Queries let us fetch data from our server, GraphQL Mutations allow us to modify our server held data.

To add a mutation to our GraphQL endpoint, we start by defining the mutation in our GraphQL Schema much like we did with queries. We’ll define a `createMessage` mutation that will enable users to send a new message to a Group:
```
type Mutation {
  # create a new message
  # text is the message text
  # userId is the id of the user sending the message
  # groupId is the id of the group receiving the message
  createMessage(text: String!, userId: Int!, groupId: Int!): Message
}
```
GraphQL Mutations are written nearly identically like GraphQL Queries. For now, we will require a `userId` parameter to identify who is creating the `Message`, but we won’t need this field once we implement authentication in a future tutorial.

Let’s update our Schema in `server/data/schema.js` to include the mutation:

[{]: <helper> (diffStep 4.1)

#### [Step 4.1: Add Mutations to Schema](https://github.com/srtucker22/chatty/commit/19da6c5)

##### Changed server&#x2F;data&#x2F;schema.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊44┊44┊    group(id: Int!): Group
 ┊45┊45┊  }
 ┊46┊46┊
<b>+┊  ┊47┊  type Mutation {</b>
<b>+┊  ┊48┊    # send a message to a group</b>
<b>+┊  ┊49┊    createMessage(</b>
<b>+┊  ┊50┊      text: String!, userId: Int!, groupId: Int!</b>
<b>+┊  ┊51┊    ): Message</b>
<b>+┊  ┊52┊  }</b>
<b>+┊  ┊53┊  </b>
 ┊47┊54┊  schema {
 ┊48┊55┊    query: Query
<b>+┊  ┊56┊    mutation: Mutation</b>
 ┊49┊57┊  }
 ┊50┊58┊&#x60;;
</pre>

[}]: #

We also need to modify our resolvers to handle our new mutation. We’ll modify `server/data/resolvers.js` as follows:

[{]: <helper> (diffStep 4.2)

#### [Step 4.2: Add Mutations to Resolvers](https://github.com/srtucker22/chatty/commit/ce16c4d)

##### Changed server&#x2F;data&#x2F;resolvers.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊18┊18┊      return User.findOne({ where: args });
 ┊19┊19┊    },
 ┊20┊20┊  },
<b>+┊  ┊21┊  Mutation: {</b>
<b>+┊  ┊22┊    createMessage(_, { text, userId, groupId }) {</b>
<b>+┊  ┊23┊      return Message.create({</b>
<b>+┊  ┊24┊        userId,</b>
<b>+┊  ┊25┊        text,</b>
<b>+┊  ┊26┊        groupId,</b>
<b>+┊  ┊27┊      });</b>
<b>+┊  ┊28┊    },</b>
<b>+┊  ┊29┊  },</b>
 ┊21┊30┊  Group: {
 ┊22┊31┊    users(group) {
 ┊23┊32┊      return group.getUsers();
</pre>

[}]: #

That’s it! When a client uses `createMessage`, the resolver will use the `Message` model passed by our connector and call `Message.create` with arguments from the mutation. The `Message.create` function returns a Promise that will resolve with the newly created `Message`.

We can easily test our newly minted `createMessage` mutation in GraphIQL to make sure everything works: ![Create Message Img](https://github.com/srtucker22/chatty/blob/master/.tortilla/media/step4-2.png)

# Designing the Input
Wow, that was way faster than when we added queries! All the heavy lifting we did in the first 3 parts of this series is starting to pay off….

Now that our server allows clients to create messages, we can build that functionality into our React Native client. First, we’ll start by creating a new component `MessageInput` where our users will be able to input their messages.

For this component, let's use **cool icons**. [`react-native-vector-icons`](https://github.com/oblador/react-native-vector-icons) is the goto package for adding icons to React Native. Please follow the instructions in the [`react-native-vector-icons` README](https://github.com/oblador/react-native-vector-icons) before moving onto the next step.

```
# make sure you're adding this package in the client folder!!!
cd client

npm i react-native-vector-icons
react-native link
# this is not enough to install icons!!! PLEASE FOLLOW THE INSTRUCTIONS IN THE README TO PROPERLY INSTALL ICONS!
```
After completing the steps in the README to install icons, we can start putting together the `MessageInput` component in a new file `client/src/components/message-input.component.js`:

[{]: <helper> (diffStep 4.3 files="client/src/components/message-input.component.js")

#### [Step 4.3: Create MessageInput](https://github.com/srtucker22/chatty/commit/bc17e3f)

##### Added client&#x2F;src&#x2F;components&#x2F;message-input.component.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊  ┊ 1┊import React, { Component } from &#x27;react&#x27;;</b>
<b>+┊  ┊ 2┊import PropTypes from &#x27;prop-types&#x27;;</b>
<b>+┊  ┊ 3┊import {</b>
<b>+┊  ┊ 4┊  StyleSheet,</b>
<b>+┊  ┊ 5┊  TextInput,</b>
<b>+┊  ┊ 6┊  View,</b>
<b>+┊  ┊ 7┊} from &#x27;react-native&#x27;;</b>
<b>+┊  ┊ 8┊</b>
<b>+┊  ┊ 9┊import Icon from &#x27;react-native-vector-icons/FontAwesome&#x27;;</b>
<b>+┊  ┊10┊</b>
<b>+┊  ┊11┊const styles &#x3D; StyleSheet.create({</b>
<b>+┊  ┊12┊  container: {</b>
<b>+┊  ┊13┊    alignSelf: &#x27;flex-end&#x27;,</b>
<b>+┊  ┊14┊    backgroundColor: &#x27;#f5f1ee&#x27;,</b>
<b>+┊  ┊15┊    borderColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊  ┊16┊    borderTopWidth: 1,</b>
<b>+┊  ┊17┊    flexDirection: &#x27;row&#x27;,</b>
<b>+┊  ┊18┊  },</b>
<b>+┊  ┊19┊  inputContainer: {</b>
<b>+┊  ┊20┊    flex: 1,</b>
<b>+┊  ┊21┊    paddingHorizontal: 12,</b>
<b>+┊  ┊22┊    paddingVertical: 6,</b>
<b>+┊  ┊23┊  },</b>
<b>+┊  ┊24┊  input: {</b>
<b>+┊  ┊25┊    backgroundColor: &#x27;white&#x27;,</b>
<b>+┊  ┊26┊    borderColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊  ┊27┊    borderRadius: 15,</b>
<b>+┊  ┊28┊    borderWidth: 1,</b>
<b>+┊  ┊29┊    color: &#x27;black&#x27;,</b>
<b>+┊  ┊30┊    height: 32,</b>
<b>+┊  ┊31┊    paddingHorizontal: 8,</b>
<b>+┊  ┊32┊  },</b>
<b>+┊  ┊33┊  sendButtonContainer: {</b>
<b>+┊  ┊34┊    paddingRight: 12,</b>
<b>+┊  ┊35┊    paddingVertical: 6,</b>
<b>+┊  ┊36┊  },</b>
<b>+┊  ┊37┊  sendButton: {</b>
<b>+┊  ┊38┊    height: 32,</b>
<b>+┊  ┊39┊    width: 32,</b>
<b>+┊  ┊40┊  },</b>
<b>+┊  ┊41┊  iconStyle: {</b>
<b>+┊  ┊42┊    marginRight: 0, // default is 12</b>
<b>+┊  ┊43┊  },</b>
<b>+┊  ┊44┊});</b>
<b>+┊  ┊45┊</b>
<b>+┊  ┊46┊const sendButton &#x3D; send &#x3D;&gt; (</b>
<b>+┊  ┊47┊  &lt;Icon.Button</b>
<b>+┊  ┊48┊    backgroundColor&#x3D;{&#x27;blue&#x27;}</b>
<b>+┊  ┊49┊    borderRadius&#x3D;{16}</b>
<b>+┊  ┊50┊    color&#x3D;{&#x27;white&#x27;}</b>
<b>+┊  ┊51┊    iconStyle&#x3D;{styles.iconStyle}</b>
<b>+┊  ┊52┊    name&#x3D;&quot;send&quot;</b>
<b>+┊  ┊53┊    onPress&#x3D;{send}</b>
<b>+┊  ┊54┊    size&#x3D;{16}</b>
<b>+┊  ┊55┊    style&#x3D;{styles.sendButton}</b>
<b>+┊  ┊56┊  /&gt;</b>
<b>+┊  ┊57┊);</b>
<b>+┊  ┊58┊</b>
<b>+┊  ┊59┊class MessageInput extends Component {</b>
<b>+┊  ┊60┊  constructor(props) {</b>
<b>+┊  ┊61┊    super(props);</b>
<b>+┊  ┊62┊    this.state &#x3D; {};</b>
<b>+┊  ┊63┊    this.send &#x3D; this.send.bind(this);</b>
<b>+┊  ┊64┊  }</b>
<b>+┊  ┊65┊</b>
<b>+┊  ┊66┊  send() {</b>
<b>+┊  ┊67┊    this.props.send(this.state.text);</b>
<b>+┊  ┊68┊    this.textInput.clear();</b>
<b>+┊  ┊69┊    this.textInput.blur();</b>
<b>+┊  ┊70┊  }</b>
<b>+┊  ┊71┊</b>
<b>+┊  ┊72┊  render() {</b>
<b>+┊  ┊73┊    return (</b>
<b>+┊  ┊74┊      &lt;View style&#x3D;{styles.container}&gt;</b>
<b>+┊  ┊75┊        &lt;View style&#x3D;{styles.inputContainer}&gt;</b>
<b>+┊  ┊76┊          &lt;TextInput</b>
<b>+┊  ┊77┊            ref&#x3D;{(ref) &#x3D;&gt; { this.textInput &#x3D; ref; }}</b>
<b>+┊  ┊78┊            onChangeText&#x3D;{text &#x3D;&gt; this.setState({ text })}</b>
<b>+┊  ┊79┊            style&#x3D;{styles.input}</b>
<b>+┊  ┊80┊            placeholder&#x3D;&quot;Type your message here!&quot;</b>
<b>+┊  ┊81┊          /&gt;</b>
<b>+┊  ┊82┊        &lt;/View&gt;</b>
<b>+┊  ┊83┊        &lt;View style&#x3D;{styles.sendButtonContainer}&gt;</b>
<b>+┊  ┊84┊          {sendButton(this.send)}</b>
<b>+┊  ┊85┊        &lt;/View&gt;</b>
<b>+┊  ┊86┊      &lt;/View&gt;</b>
<b>+┊  ┊87┊    );</b>
<b>+┊  ┊88┊  }</b>
<b>+┊  ┊89┊}</b>
<b>+┊  ┊90┊</b>
<b>+┊  ┊91┊MessageInput.propTypes &#x3D; {</b>
<b>+┊  ┊92┊  send: PropTypes.func.isRequired,</b>
<b>+┊  ┊93┊};</b>
<b>+┊  ┊94┊</b>
<b>+┊  ┊95┊export default MessageInput;</b>
</pre>

[}]: #

Our `MessageInput` component is a `View` that wraps a controlled `TextInput` and an [`Icon.Button`](https://github.com/oblador/react-native-vector-icons#iconbutton-component). When the button is pressed, `props.send` will be called with the current state of the `TextInput` text and then the `TextInput` will clear. We’ve also added some styling to keep everything looking snazzy.

Let’s add `MessageInput` to the bottom of the `Messages` screen and create a placeholder `send` function:

[{]: <helper> (diffStep 4.4)

#### [Step 4.4: Add MessageInput to Messages](https://github.com/srtucker22/chatty/commit/e731ca8)

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊10┊10┊import { graphql, compose } from &#x27;react-apollo&#x27;;
 ┊11┊11┊
 ┊12┊12┊import Message from &#x27;../components/message.component&#x27;;
<b>+┊  ┊13┊import MessageInput from &#x27;../components/message-input.component&#x27;;</b>
 ┊13┊14┊import GROUP_QUERY from &#x27;../graphql/group.query&#x27;;
 ┊14┊15┊
 ┊15┊16┊const styles &#x3D; StyleSheet.create({
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊46┊47┊    };
 ┊47┊48┊
 ┊48┊49┊    this.renderItem &#x3D; this.renderItem.bind(this);
<b>+┊  ┊50┊    this.send &#x3D; this.send.bind(this);</b>
 ┊49┊51┊  }
 ┊50┊52┊
 ┊51┊53┊  componentWillReceiveProps(nextProps) {
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊65┊67┊    }
 ┊66┊68┊  }
 ┊67┊69┊
<b>+┊  ┊70┊  send(text) {</b>
<b>+┊  ┊71┊    // TODO: send the message</b>
<b>+┊  ┊72┊    console.log(&#x60;sending message: ${text}&#x60;);</b>
<b>+┊  ┊73┊  }</b>
<b>+┊  ┊74┊</b>
 ┊68┊75┊  keyExtractor &#x3D; item &#x3D;&gt; item.id.toString();
 ┊69┊76┊
 ┊70┊77┊  renderItem &#x3D; ({ item: message }) &#x3D;&gt; (
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊ 96┊103┊          renderItem&#x3D;{this.renderItem}
 ┊ 97┊104┊          ListEmptyComponent&#x3D;{&lt;View /&gt;}
 ┊ 98┊105┊        /&gt;
<b>+┊   ┊106┊        &lt;MessageInput send&#x3D;{this.send} /&gt;</b>
 ┊ 99┊107┊      &lt;/View&gt;
 ┊100┊108┊    );
 ┊101┊109┊  }
</pre>

[}]: #

It should look like this: ![Message Input Image](https://github.com/srtucker22/chatty/blob/master/.tortilla/media/step4-4.png)

But **don’t be fooled by your simulator!** This UI will break on a phone because of the keyboard: ![Broken Input Image](https://github.com/srtucker22/chatty/blob/master/.tortilla/media/step4-4-2.png)

You are not the first person to groan over this issue. For you and the many groaners out there, the wonderful devs at Facebook have your back. [`KeyboardAvoidingView`](https://facebook.github.io/react-native/docs/keyboardavoidingview.html) to the rescue!

[{]: <helper> (diffStep 4.5)

#### [Step 4.5: Add KeyboardAvoidingView](https://github.com/srtucker22/chatty/commit/51b2a4a)

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊1┊1┊import {
 ┊2┊2┊  ActivityIndicator,
 ┊3┊3┊  FlatList,
<b>+┊ ┊4┊  KeyboardAvoidingView,</b>
 ┊4┊5┊  StyleSheet,
 ┊5┊6┊  View,
 ┊6┊7┊} from &#x27;react-native&#x27;;
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊ 96┊ 97┊
 ┊ 97┊ 98┊    // render list of messages for group
 ┊ 98┊ 99┊    return (
<b>+┊   ┊100┊      &lt;KeyboardAvoidingView</b>
<b>+┊   ┊101┊        behavior&#x3D;{&#x27;position&#x27;}</b>
<b>+┊   ┊102┊        contentContainerStyle&#x3D;{styles.container}</b>
<b>+┊   ┊103┊        keyboardVerticalOffset&#x3D;{64}</b>
<b>+┊   ┊104┊        style&#x3D;{styles.container}</b>
<b>+┊   ┊105┊      &gt;</b>
 ┊100┊106┊        &lt;FlatList
 ┊101┊107┊          data&#x3D;{group.messages.slice().reverse()}
 ┊102┊108┊          keyExtractor&#x3D;{this.keyExtractor}
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊104┊110┊          ListEmptyComponent&#x3D;{&lt;View /&gt;}
 ┊105┊111┊        /&gt;
 ┊106┊112┊        &lt;MessageInput send&#x3D;{this.send} /&gt;
<b>+┊   ┊113┊      &lt;/KeyboardAvoidingView&gt;</b>
 ┊108┊114┊    );
 ┊109┊115┊  }
 ┊110┊116┊}
</pre>

[}]: #

![Fixed Input Image](https://github.com/srtucker22/chatty/blob/master/.tortilla/media/step4-5.png)

Our layout looks ready. Now let’s make it work!

# Adding GraphQL Mutations on the Client
Let’s start by defining our GraphQL Mutation like we would using GraphIQL:
```graphql
mutation createMessage($text: String!, $userId: Int!, $groupId: Int!) {
  createMessage(text: $text, userId: $userId, groupId: $groupId) {
    id
    from {
      id
      username
    }
    createdAt
    text
  }
}
```
That looks fine, but notice the `Message` fields we want to see returned look exactly like the `Message` fields we are using for `GROUP_QUERY`:
```graphql
query group($groupId: Int!) {
  group(id: $groupId) {
    id
    name
    users {
      id
      username
    }
    messages {
      id
      from {
        id
        username
      }
      createdAt
      text
    }
  }
}
```
GraphQL allows us to reuse pieces of queries and mutations with [**Fragments**](http://graphql.org/learn/queries/#fragments). We can factor out this common set of fields into a `MessageFragment` that looks like this:

[{]: <helper> (diffStep 4.6)

#### [Step 4.6: Create MessageFragment](https://github.com/srtucker22/chatty/commit/d197051)

##### Added client&#x2F;src&#x2F;graphql&#x2F;message.fragment.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊  ┊ 1┊import gql from &#x27;graphql-tag&#x27;;</b>
<b>+┊  ┊ 2┊</b>
<b>+┊  ┊ 3┊const MESSAGE_FRAGMENT &#x3D; gql&#x60;</b>
<b>+┊  ┊ 4┊  fragment MessageFragment on Message {</b>
<b>+┊  ┊ 5┊    id</b>
<b>+┊  ┊ 6┊    to {</b>
<b>+┊  ┊ 7┊      id</b>
<b>+┊  ┊ 8┊    }</b>
<b>+┊  ┊ 9┊    from {</b>
<b>+┊  ┊10┊      id</b>
<b>+┊  ┊11┊      username</b>
<b>+┊  ┊12┊    }</b>
<b>+┊  ┊13┊    createdAt</b>
<b>+┊  ┊14┊    text</b>
<b>+┊  ┊15┊  }</b>
<b>+┊  ┊16┊&#x60;;</b>
<b>+┊  ┊17┊</b>
<b>+┊  ┊18┊export default MESSAGE_FRAGMENT;</b>
</pre>

[}]: #

Now we can apply `MESSAGE_FRAGMENT` to `GROUP_QUERY` by changing our code as follows:

[{]: <helper> (diffStep 4.7)

#### [Step 4.7: Add MessageFragment to Group Query](https://github.com/srtucker22/chatty/commit/c91d503)

##### Changed client&#x2F;src&#x2F;graphql&#x2F;group.query.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊1┊1┊import gql from &#x27;graphql-tag&#x27;;
 ┊2┊2┊
<b>+┊ ┊3┊import MESSAGE_FRAGMENT from &#x27;./message.fragment&#x27;;</b>
<b>+┊ ┊4┊</b>
 ┊3┊5┊const GROUP_QUERY &#x3D; gql&#x60;
 ┊4┊6┊  query group($groupId: Int!) {
 ┊5┊7┊    group(id: $groupId) {
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊10┊12┊        username
 ┊11┊13┊      }
 ┊12┊14┊      messages {
<b>+┊  ┊15┊        ... MessageFragment</b>
 ┊20┊16┊      }
 ┊21┊17┊    }
 ┊22┊18┊  }
<b>+┊  ┊19┊  ${MESSAGE_FRAGMENT}</b>
 ┊23┊20┊&#x60;;
 ┊24┊21┊
 ┊25┊22┊export default GROUP_QUERY;
</pre>

[}]: #

Let’s also write our `createMessage` mutation using `messageFragment` in a new file `client/src/graphql/create-message.mutation.js`:

[{]: <helper> (diffStep 4.8)

#### [Step 4.8: Create CREATE_MESSAGE_MUTATION](https://github.com/srtucker22/chatty/commit/9199c34)

##### Added client&#x2F;src&#x2F;graphql&#x2F;create-message.mutation.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊  ┊ 1┊import gql from &#x27;graphql-tag&#x27;;</b>
<b>+┊  ┊ 2┊</b>
<b>+┊  ┊ 3┊import MESSAGE_FRAGMENT from &#x27;./message.fragment&#x27;;</b>
<b>+┊  ┊ 4┊</b>
<b>+┊  ┊ 5┊const CREATE_MESSAGE_MUTATION &#x3D; gql&#x60;</b>
<b>+┊  ┊ 6┊  mutation createMessage($text: String!, $userId: Int!, $groupId: Int!) {</b>
<b>+┊  ┊ 7┊    createMessage(text: $text, userId: $userId, groupId: $groupId) {</b>
<b>+┊  ┊ 8┊      ... MessageFragment</b>
<b>+┊  ┊ 9┊    }</b>
<b>+┊  ┊10┊  }</b>
<b>+┊  ┊11┊  ${MESSAGE_FRAGMENT}</b>
<b>+┊  ┊12┊&#x60;;</b>
<b>+┊  ┊13┊</b>
<b>+┊  ┊14┊export default CREATE_MESSAGE_MUTATION;</b>
</pre>

[}]: #

Now all we have to do is plug our mutation into our `Messages` component using the `graphql` module from `react-apollo`. Before we connect everything, let’s see what a mutation call with the `graphql` module looks like:
```js
const createMessage = graphql(CREATE_MESSAGE_MUTATION, {
  props: ({ ownProps, mutate }) => ({
    createMessage: ({ text, userId, groupId }) =>
      mutate({
        variables: { text, userId, groupId },
      }),
  }),
});
```
Just like with a GraphQL Query, we first pass our mutation to `graphql`, followed by an Object with configuration params. The `props` param accepts a function with named arguments including `ownProps` (the components current props) and `mutate`. This function should return an Object with the name of the function that we plan to call inside our component, which executes `mutate` with the variables we wish to pass. If that sounds complicated, it’s because it is. Kudos to the Meteor team for putting it together though, because it’s actually some very clever code.

At the end of the day, once you write your first mutation, it’s really mostly a matter of copy/paste and changing the names of the variables.

Okay, so let’s put it all together in `messages.screen.js`:

[{]: <helper> (diffStep 4.9)

#### [Step 4.9: Add CREATE_MESSAGE_MUTATION to Messages](https://github.com/srtucker22/chatty/commit/3b66d67)

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊13┊13┊import Message from &#x27;../components/message.component&#x27;;
 ┊14┊14┊import MessageInput from &#x27;../components/message-input.component&#x27;;
 ┊15┊15┊import GROUP_QUERY from &#x27;../graphql/group.query&#x27;;
<b>+┊  ┊16┊import CREATE_MESSAGE_MUTATION from &#x27;../graphql/create-message.mutation&#x27;;</b>
 ┊16┊17┊
 ┊17┊18┊const styles &#x3D; StyleSheet.create({
 ┊18┊19┊  container: {
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊69┊70┊  }
 ┊70┊71┊
 ┊71┊72┊  send(text) {
<b>+┊  ┊73┊    this.props.createMessage({</b>
<b>+┊  ┊74┊      groupId: this.props.navigation.state.params.groupId,</b>
<b>+┊  ┊75┊      userId: 1, // faking the user for now</b>
<b>+┊  ┊76┊      text,</b>
<b>+┊  ┊77┊    });</b>
 ┊74┊78┊  }
 ┊75┊79┊
 ┊76┊80┊  keyExtractor &#x3D; item &#x3D;&gt; item.id.toString();
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊116┊120┊}
 ┊117┊121┊
 ┊118┊122┊Messages.propTypes &#x3D; {
<b>+┊   ┊123┊  createMessage: PropTypes.func,</b>
<b>+┊   ┊124┊  navigation: PropTypes.shape({</b>
<b>+┊   ┊125┊    state: PropTypes.shape({</b>
<b>+┊   ┊126┊      params: PropTypes.shape({</b>
<b>+┊   ┊127┊        groupId: PropTypes.number,</b>
<b>+┊   ┊128┊      }),</b>
<b>+┊   ┊129┊    }),</b>
<b>+┊   ┊130┊  }),</b>
 ┊119┊131┊  group: PropTypes.shape({
 ┊120┊132┊    messages: PropTypes.array,
 ┊121┊133┊    users: PropTypes.array,
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊134┊146┊  }),
 ┊135┊147┊});
 ┊136┊148┊
<b>+┊   ┊149┊const createMessageMutation &#x3D; graphql(CREATE_MESSAGE_MUTATION, {</b>
<b>+┊   ┊150┊  props: ({ mutate }) &#x3D;&gt; ({</b>
<b>+┊   ┊151┊    createMessage: ({ text, userId, groupId }) &#x3D;&gt;</b>
<b>+┊   ┊152┊      mutate({</b>
<b>+┊   ┊153┊        variables: { text, userId, groupId },</b>
<b>+┊   ┊154┊      }),</b>
<b>+┊   ┊155┊  }),</b>
<b>+┊   ┊156┊});</b>
<b>+┊   ┊157┊</b>
 ┊137┊158┊export default compose(
 ┊138┊159┊  groupQuery,
<b>+┊   ┊160┊  createMessageMutation,</b>
 ┊139┊161┊)(Messages);
</pre>

[}]: #

By attaching `createMessage` with `compose`, we attach a `createMessage` function to the component’s `props`. We call `props.createMessage` in `send` with the required variables (we’ll keep faking the user for now). When the user presses the send button, this method will get called and the mutation should execute.

Let’s run the app and see what happens: ![Send Fail Gif](https://github.com/srtucker22/chatty/blob/master/.tortilla/media/step4-9.gif)

What went wrong? Well technically nothing went wrong. Our mutation successfully executed, but we’re not seeing our message pop up. Why? **Running a mutation doesn’t automatically update our queries with new data!** If we were to refresh the page, we’d actually see our message. This issue only arrises when we are adding or removing data with our mutation.

To overcome this challenge, `react-apollo` lets us declare a property `update` within the argument we pass to mutate. In `update`, we specify which queries should update after the mutation executes and how the data will transform.

Our modified `createMessage` should look like this:

[{]: <helper> (diffStep "4.10")

#### [Step 4.10: Add update to mutation](https://github.com/srtucker22/chatty/commit/228c5e7)

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊151┊151┊    createMessage: ({ text, userId, groupId }) &#x3D;&gt;
 ┊152┊152┊      mutate({
 ┊153┊153┊        variables: { text, userId, groupId },
<b>+┊   ┊154┊        update: (store, { data: { createMessage } }) &#x3D;&gt; {</b>
<b>+┊   ┊155┊          // Read the data from our cache for this query.</b>
<b>+┊   ┊156┊          const groupData &#x3D; store.readQuery({</b>
<b>+┊   ┊157┊            query: GROUP_QUERY,</b>
<b>+┊   ┊158┊            variables: {</b>
<b>+┊   ┊159┊              groupId,</b>
<b>+┊   ┊160┊            },</b>
<b>+┊   ┊161┊          });</b>
<b>+┊   ┊162┊</b>
<b>+┊   ┊163┊          // Add our message from the mutation to the end.</b>
<b>+┊   ┊164┊          groupData.group.messages.unshift(createMessage);</b>
<b>+┊   ┊165┊</b>
<b>+┊   ┊166┊          // Write our data back to the cache.</b>
<b>+┊   ┊167┊          store.writeQuery({</b>
<b>+┊   ┊168┊            query: GROUP_QUERY,</b>
<b>+┊   ┊169┊            variables: {</b>
<b>+┊   ┊170┊              groupId,</b>
<b>+┊   ┊171┊            },</b>
<b>+┊   ┊172┊            data: groupData,</b>
<b>+┊   ┊173┊          });</b>
<b>+┊   ┊174┊        },</b>
 ┊154┊175┊      }),
<b>+┊   ┊176┊</b>
 ┊155┊177┊  }),
 ┊156┊178┊});
</pre>

[}]: #

In `update`, we first retrieve the existing data for the query we want to update (`GROUP_QUERY`) along with the specific variables we passed to that query. This data comes to us from our Redux store of Apollo data. We check to see if the new `Message` returned from `createMessage` already exists (in case of race conditions down the line), and then update the previous query result by sticking the new message in front. We then use this modified data object and rewrite the results to the Apollo store with `store.writeQuery`, being sure to pass all the variables associated with our query. This will force `props` to change reference and the component to rerender. ![Fixed Send Gif](https://github.com/srtucker22/chatty/blob/master/.tortilla/media/step4-10.gif)

# Optimistic UI
### But wait! There’s more!
`update` will currently only update the query after the mutation succeeds and a response is sent back on the server. But we don’t want to wait till the server returns data  —  we crave instant gratification! If a user with shoddy internet tried to send a message and it didn’t show up right away, they’d probably try and send the message again and again and end up sending the message multiple times… and then they’d yell at customer support!

**Optimistic UI** is our weapon for protecting customer support. We know the shape of the data we expect to receive from the server, so why not fake it until we get a response? `react-apollo` lets us accomplish this by adding an `optimisticResponse` parameter to mutate. In our case it looks like this:

[{]: <helper> (diffStep 4.11)

#### [Step 4.11: Add optimisticResponse to mutation](https://github.com/srtucker22/chatty/commit/f3f8ac9)

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊151┊151┊    createMessage: ({ text, userId, groupId }) &#x3D;&gt;
 ┊152┊152┊      mutate({
 ┊153┊153┊        variables: { text, userId, groupId },
<b>+┊   ┊154┊        optimisticResponse: {</b>
<b>+┊   ┊155┊          __typename: &#x27;Mutation&#x27;,</b>
<b>+┊   ┊156┊          createMessage: {</b>
<b>+┊   ┊157┊            __typename: &#x27;Message&#x27;,</b>
<b>+┊   ┊158┊            id: -1, // don&#x27;t know id yet, but it doesn&#x27;t matter</b>
<b>+┊   ┊159┊            text, // we know what the text will be</b>
<b>+┊   ┊160┊            createdAt: new Date().toISOString(), // the time is now!</b>
<b>+┊   ┊161┊            from: {</b>
<b>+┊   ┊162┊              __typename: &#x27;User&#x27;,</b>
<b>+┊   ┊163┊              id: 1, // still faking the user</b>
<b>+┊   ┊164┊              username: &#x27;Justyn.Kautzer&#x27;, // still faking the user</b>
<b>+┊   ┊165┊            },</b>
<b>+┊   ┊166┊            to: {</b>
<b>+┊   ┊167┊              __typename: &#x27;Group&#x27;,</b>
<b>+┊   ┊168┊              id: groupId,</b>
<b>+┊   ┊169┊            },</b>
<b>+┊   ┊170┊          },</b>
<b>+┊   ┊171┊        },</b>
 ┊154┊172┊        update: (store, { data: { createMessage } }) &#x3D;&gt; {
 ┊155┊173┊          // Read the data from our cache for this query.
 ┊156┊174┊          const groupData &#x3D; store.readQuery({
</pre>

[}]: #

The Object returned from `optimisticResponse` is what the data should look like from our server when the mutation succeeds. We need to specify the `__typename` for all  values in our optimistic response just like our server would. Even though we don’t know all values for all fields, we know enough to populate the ones that will show up in the UI, like the text, user, and message creation time. This will essentially be a placeholder until the server responds.

Let’s also modify our UI a bit so that our `FlatList` scrolls to the bottom when we send a message as soon as we receive new data:

[{]: <helper> (diffStep 4.12)

#### [Step 4.12: Add scrollToEnd to Messages after send](https://github.com/srtucker22/chatty/commit/2d869a6)

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊74┊74┊      groupId: this.props.navigation.state.params.groupId,
 ┊75┊75┊      userId: 1, // faking the user for now
 ┊76┊76┊      text,
<b>+┊  ┊77┊    }).then(() &#x3D;&gt; {</b>
<b>+┊  ┊78┊      this.flatList.scrollToEnd({ animated: true });</b>
 ┊77┊79┊    });
 ┊78┊80┊  }
 ┊79┊81┊
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊108┊110┊        style&#x3D;{styles.container}
 ┊109┊111┊      &gt;
 ┊110┊112┊        &lt;FlatList
<b>+┊   ┊113┊          ref&#x3D;{(ref) &#x3D;&gt; { this.flatList &#x3D; ref; }}</b>
 ┊111┊114┊          data&#x3D;{group.messages.slice().reverse()}
 ┊112┊115┊          keyExtractor&#x3D;{this.keyExtractor}
 ┊113┊116┊          renderItem&#x3D;{this.renderItem}
</pre>

[}]: #

![Scroll to Bottom Gif](https://github.com/srtucker22/chatty/blob/master/.tortilla/media/step4-12.gif)

### 🔥🔥🔥!!!

# **YOUR CHALLENGE**
First, let’s take a break. We’ve definitely earned it.

Now that we’re comfortable using GraphQL Queries and Mutations and some tricky stuff in React Native, we can do most of the things we need to do for most basic applications. In fact, there are a number of Chatty features that we can already implement without knowing much else. This post is already plenty long, but there are features left to be built. So with that said, I like to suggest that you try to complete the following features on your own before we move on:

1. Add GraphQL Mutations on our server for creating, modifying, and deleting `Groups`
2. Add new Screens to our React Native app for creating, modifying, and deleting `Groups`
3. Build GraphQL Queries and Mutations for our new Screens and connect them using `react-apollo`
4. Include `update` for these new mutations where necessary

If you want to see some UI or you want a hint or you don’t wanna write any code, that’s cool too! Below is some code with these features added. ![Groups Gif](https://github.com/srtucker22/chatty/blob/master/.tortilla/media/step4-13.gif)

[{]: <helper> (diffStep 4.13)

#### [Step 4.13: Add Group Mutations and Screens](https://github.com/srtucker22/chatty/commit/948d661)

##### Changed client&#x2F;package.json
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊15┊15┊		&quot;apollo-link-redux&quot;: &quot;^0.2.1&quot;,
 ┊16┊16┊		&quot;graphql&quot;: &quot;^0.12.3&quot;,
 ┊17┊17┊		&quot;graphql-tag&quot;: &quot;^2.4.2&quot;,
<b>+┊  ┊18┊		&quot;immutability-helper&quot;: &quot;^2.6.4&quot;,</b>
 ┊18┊19┊		&quot;lodash&quot;: &quot;^4.17.5&quot;,
 ┊19┊20┊		&quot;moment&quot;: &quot;^2.20.1&quot;,
 ┊20┊21┊		&quot;prop-types&quot;: &quot;^15.6.0&quot;,
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊22┊23┊		&quot;react&quot;: &quot;16.4.1&quot;,
 ┊23┊24┊		&quot;react-apollo&quot;: &quot;^2.0.4&quot;,
 ┊24┊25┊		&quot;react-native&quot;: &quot;0.56.0&quot;,
<b>+┊  ┊26┊		&quot;react-native-alpha-listview&quot;: &quot;^0.2.1&quot;,</b>
 ┊25┊27┊		&quot;react-native-vector-icons&quot;: &quot;^4.6.0&quot;,
 ┊26┊28┊		&quot;react-navigation&quot;: &quot;^1.0.3&quot;,
 ┊27┊29┊		&quot;react-navigation-redux-helpers&quot;: &quot;^1.1.2&quot;,
</pre>

##### Added client&#x2F;src&#x2F;components&#x2F;selected-user-list.component.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊   ┊  1┊import React, { Component } from &#x27;react&#x27;;</b>
<b>+┊   ┊  2┊import PropTypes from &#x27;prop-types&#x27;;</b>
<b>+┊   ┊  3┊import {</b>
<b>+┊   ┊  4┊  FlatList,</b>
<b>+┊   ┊  5┊  Image,</b>
<b>+┊   ┊  6┊  StyleSheet,</b>
<b>+┊   ┊  7┊  Text,</b>
<b>+┊   ┊  8┊  TouchableOpacity,</b>
<b>+┊   ┊  9┊  View,</b>
<b>+┊   ┊ 10┊} from &#x27;react-native&#x27;;</b>
<b>+┊   ┊ 11┊import Icon from &#x27;react-native-vector-icons/FontAwesome&#x27;;</b>
<b>+┊   ┊ 12┊</b>
<b>+┊   ┊ 13┊const styles &#x3D; StyleSheet.create({</b>
<b>+┊   ┊ 14┊  list: {</b>
<b>+┊   ┊ 15┊    paddingVertical: 8,</b>
<b>+┊   ┊ 16┊  },</b>
<b>+┊   ┊ 17┊  itemContainer: {</b>
<b>+┊   ┊ 18┊    alignItems: &#x27;center&#x27;,</b>
<b>+┊   ┊ 19┊    paddingHorizontal: 12,</b>
<b>+┊   ┊ 20┊  },</b>
<b>+┊   ┊ 21┊  itemIcon: {</b>
<b>+┊   ┊ 22┊    alignItems: &#x27;center&#x27;,</b>
<b>+┊   ┊ 23┊    backgroundColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊   ┊ 24┊    borderColor: &#x27;white&#x27;,</b>
<b>+┊   ┊ 25┊    borderRadius: 10,</b>
<b>+┊   ┊ 26┊    borderWidth: 2,</b>
<b>+┊   ┊ 27┊    flexDirection: &#x27;row&#x27;,</b>
<b>+┊   ┊ 28┊    height: 20,</b>
<b>+┊   ┊ 29┊    justifyContent: &#x27;center&#x27;,</b>
<b>+┊   ┊ 30┊    position: &#x27;absolute&#x27;,</b>
<b>+┊   ┊ 31┊    right: -3,</b>
<b>+┊   ┊ 32┊    top: -3,</b>
<b>+┊   ┊ 33┊    width: 20,</b>
<b>+┊   ┊ 34┊  },</b>
<b>+┊   ┊ 35┊  itemImage: {</b>
<b>+┊   ┊ 36┊    borderRadius: 27,</b>
<b>+┊   ┊ 37┊    height: 54,</b>
<b>+┊   ┊ 38┊    width: 54,</b>
<b>+┊   ┊ 39┊  },</b>
<b>+┊   ┊ 40┊});</b>
<b>+┊   ┊ 41┊</b>
<b>+┊   ┊ 42┊export class SelectedUserListItem extends Component {</b>
<b>+┊   ┊ 43┊  constructor(props) {</b>
<b>+┊   ┊ 44┊    super(props);</b>
<b>+┊   ┊ 45┊</b>
<b>+┊   ┊ 46┊    this.remove &#x3D; this.remove.bind(this);</b>
<b>+┊   ┊ 47┊  }</b>
<b>+┊   ┊ 48┊</b>
<b>+┊   ┊ 49┊  remove() {</b>
<b>+┊   ┊ 50┊    this.props.remove(this.props.user);</b>
<b>+┊   ┊ 51┊  }</b>
<b>+┊   ┊ 52┊</b>
<b>+┊   ┊ 53┊  render() {</b>
<b>+┊   ┊ 54┊    const { username } &#x3D; this.props.user;</b>
<b>+┊   ┊ 55┊</b>
<b>+┊   ┊ 56┊    return (</b>
<b>+┊   ┊ 57┊      &lt;View</b>
<b>+┊   ┊ 58┊        style&#x3D;{styles.itemContainer}</b>
<b>+┊   ┊ 59┊      &gt;</b>
<b>+┊   ┊ 60┊        &lt;View&gt;</b>
<b>+┊   ┊ 61┊          &lt;Image</b>
<b>+┊   ┊ 62┊            style&#x3D;{styles.itemImage}</b>
<b>+┊   ┊ 63┊            source&#x3D;{{ uri: &#x27;https://reactjs.org/logo-og.png&#x27; }}</b>
<b>+┊   ┊ 64┊          /&gt;</b>
<b>+┊   ┊ 65┊          &lt;TouchableOpacity onPress&#x3D;{this.remove} style&#x3D;{styles.itemIcon}&gt;</b>
<b>+┊   ┊ 66┊            &lt;Icon</b>
<b>+┊   ┊ 67┊              color&#x3D;&quot;white&quot;</b>
<b>+┊   ┊ 68┊              name&#x3D;&quot;times&quot;</b>
<b>+┊   ┊ 69┊              size&#x3D;{12}</b>
<b>+┊   ┊ 70┊            /&gt;</b>
<b>+┊   ┊ 71┊          &lt;/TouchableOpacity&gt;</b>
<b>+┊   ┊ 72┊        &lt;/View&gt;</b>
<b>+┊   ┊ 73┊        &lt;Text&gt;{username}&lt;/Text&gt;</b>
<b>+┊   ┊ 74┊      &lt;/View&gt;</b>
<b>+┊   ┊ 75┊    );</b>
<b>+┊   ┊ 76┊  }</b>
<b>+┊   ┊ 77┊}</b>
<b>+┊   ┊ 78┊SelectedUserListItem.propTypes &#x3D; {</b>
<b>+┊   ┊ 79┊  user: PropTypes.shape({</b>
<b>+┊   ┊ 80┊    id: PropTypes.number,</b>
<b>+┊   ┊ 81┊    username: PropTypes.string,</b>
<b>+┊   ┊ 82┊  }),</b>
<b>+┊   ┊ 83┊  remove: PropTypes.func,</b>
<b>+┊   ┊ 84┊};</b>
<b>+┊   ┊ 85┊</b>
<b>+┊   ┊ 86┊class SelectedUserList extends Component {</b>
<b>+┊   ┊ 87┊  constructor(props) {</b>
<b>+┊   ┊ 88┊    super(props);</b>
<b>+┊   ┊ 89┊</b>
<b>+┊   ┊ 90┊    this.renderItem &#x3D; this.renderItem.bind(this);</b>
<b>+┊   ┊ 91┊  }</b>
<b>+┊   ┊ 92┊</b>
<b>+┊   ┊ 93┊  keyExtractor &#x3D; item &#x3D;&gt; item.id.toString();</b>
<b>+┊   ┊ 94┊</b>
<b>+┊   ┊ 95┊  renderItem({ item: user }) {</b>
<b>+┊   ┊ 96┊    return (</b>
<b>+┊   ┊ 97┊      &lt;SelectedUserListItem user&#x3D;{user} remove&#x3D;{this.props.remove} /&gt;</b>
<b>+┊   ┊ 98┊    );</b>
<b>+┊   ┊ 99┊  }</b>
<b>+┊   ┊100┊</b>
<b>+┊   ┊101┊  render() {</b>
<b>+┊   ┊102┊    return (</b>
<b>+┊   ┊103┊      &lt;FlatList</b>
<b>+┊   ┊104┊        data&#x3D;{this.props.data}</b>
<b>+┊   ┊105┊        keyExtractor&#x3D;{this.keyExtractor}</b>
<b>+┊   ┊106┊        renderItem&#x3D;{this.renderItem}</b>
<b>+┊   ┊107┊        horizontal</b>
<b>+┊   ┊108┊        style&#x3D;{styles.list}</b>
<b>+┊   ┊109┊      /&gt;</b>
<b>+┊   ┊110┊    );</b>
<b>+┊   ┊111┊  }</b>
<b>+┊   ┊112┊}</b>
<b>+┊   ┊113┊SelectedUserList.propTypes &#x3D; {</b>
<b>+┊   ┊114┊  data: PropTypes.arrayOf(PropTypes.object),</b>
<b>+┊   ┊115┊  remove: PropTypes.func,</b>
<b>+┊   ┊116┊};</b>
<b>+┊   ┊117┊</b>
<b>+┊   ┊118┊export default SelectedUserList;</b>
</pre>

##### Added client&#x2F;src&#x2F;graphql&#x2F;create-group.mutation.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊  ┊ 1┊import gql from &#x27;graphql-tag&#x27;;</b>
<b>+┊  ┊ 2┊</b>
<b>+┊  ┊ 3┊const CREATE_GROUP_MUTATION &#x3D; gql&#x60;</b>
<b>+┊  ┊ 4┊  mutation createGroup($name: String!, $userIds: [Int!], $userId: Int!) {</b>
<b>+┊  ┊ 5┊    createGroup(name: $name, userIds: $userIds, userId: $userId) {</b>
<b>+┊  ┊ 6┊      id</b>
<b>+┊  ┊ 7┊      name</b>
<b>+┊  ┊ 8┊      users {</b>
<b>+┊  ┊ 9┊        id</b>
<b>+┊  ┊10┊      }</b>
<b>+┊  ┊11┊    }</b>
<b>+┊  ┊12┊  }</b>
<b>+┊  ┊13┊&#x60;;</b>
<b>+┊  ┊14┊</b>
<b>+┊  ┊15┊export default CREATE_GROUP_MUTATION;</b>
</pre>

##### Added client&#x2F;src&#x2F;graphql&#x2F;delete-group.mutation.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊  ┊ 1┊import gql from &#x27;graphql-tag&#x27;;</b>
<b>+┊  ┊ 2┊</b>
<b>+┊  ┊ 3┊const DELETE_GROUP_MUTATION &#x3D; gql&#x60;</b>
<b>+┊  ┊ 4┊  mutation deleteGroup($id: Int!) {</b>
<b>+┊  ┊ 5┊    deleteGroup(id: $id) {</b>
<b>+┊  ┊ 6┊      id</b>
<b>+┊  ┊ 7┊    }</b>
<b>+┊  ┊ 8┊  }</b>
<b>+┊  ┊ 9┊&#x60;;</b>
<b>+┊  ┊10┊</b>
<b>+┊  ┊11┊export default DELETE_GROUP_MUTATION;</b>
</pre>

##### Added client&#x2F;src&#x2F;graphql&#x2F;leave-group.mutation.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊  ┊ 1┊import gql from &#x27;graphql-tag&#x27;;</b>
<b>+┊  ┊ 2┊</b>
<b>+┊  ┊ 3┊const LEAVE_GROUP_MUTATION &#x3D; gql&#x60;</b>
<b>+┊  ┊ 4┊  mutation leaveGroup($id: Int!, $userId: Int!) {</b>
<b>+┊  ┊ 5┊    leaveGroup(id: $id, userId: $userId) {</b>
<b>+┊  ┊ 6┊      id</b>
<b>+┊  ┊ 7┊    }</b>
<b>+┊  ┊ 8┊  }</b>
<b>+┊  ┊ 9┊&#x60;;</b>
<b>+┊  ┊10┊</b>
<b>+┊  ┊11┊export default LEAVE_GROUP_MUTATION;</b>
</pre>

##### Changed client&#x2F;src&#x2F;graphql&#x2F;user.query.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊11┊11┊        id
 ┊12┊12┊        name
 ┊13┊13┊      }
<b>+┊  ┊14┊      friends {</b>
<b>+┊  ┊15┊        id</b>
<b>+┊  ┊16┊        username</b>
<b>+┊  ┊17┊      }</b>
 ┊14┊18┊    }
 ┊15┊19┊  }
 ┊16┊20┊&#x60;;
</pre>

##### Changed client&#x2F;src&#x2F;navigation.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊10┊10┊
 ┊11┊11┊import Groups from &#x27;./screens/groups.screen&#x27;;
 ┊12┊12┊import Messages from &#x27;./screens/messages.screen&#x27;;
<b>+┊  ┊13┊import FinalizeGroup from &#x27;./screens/finalize-group.screen&#x27;;</b>
<b>+┊  ┊14┊import GroupDetails from &#x27;./screens/group-details.screen&#x27;;</b>
<b>+┊  ┊15┊import NewGroup from &#x27;./screens/new-group.screen&#x27;;</b>
 ┊13┊16┊
 ┊14┊17┊const styles &#x3D; StyleSheet.create({
 ┊15┊18┊  container: {
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊47┊50┊const AppNavigator &#x3D; StackNavigator({
 ┊48┊51┊  Main: { screen: MainScreenNavigator },
 ┊49┊52┊  Messages: { screen: Messages },
<b>+┊  ┊53┊  GroupDetails: { screen: GroupDetails },</b>
<b>+┊  ┊54┊  NewGroup: { screen: NewGroup },</b>
<b>+┊  ┊55┊  FinalizeGroup: { screen: FinalizeGroup },</b>
 ┊50┊56┊}, {
 ┊51┊57┊  mode: &#x27;modal&#x27;,
 ┊52┊58┊});
</pre>

##### Added client&#x2F;src&#x2F;screens&#x2F;finalize-group.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊   ┊  1┊import { _ } from &#x27;lodash&#x27;;</b>
<b>+┊   ┊  2┊import React, { Component } from &#x27;react&#x27;;</b>
<b>+┊   ┊  3┊import PropTypes from &#x27;prop-types&#x27;;</b>
<b>+┊   ┊  4┊import {</b>
<b>+┊   ┊  5┊  Alert,</b>
<b>+┊   ┊  6┊  Button,</b>
<b>+┊   ┊  7┊  Image,</b>
<b>+┊   ┊  8┊  StyleSheet,</b>
<b>+┊   ┊  9┊  Text,</b>
<b>+┊   ┊ 10┊  TextInput,</b>
<b>+┊   ┊ 11┊  TouchableOpacity,</b>
<b>+┊   ┊ 12┊  View,</b>
<b>+┊   ┊ 13┊} from &#x27;react-native&#x27;;</b>
<b>+┊   ┊ 14┊import { graphql, compose } from &#x27;react-apollo&#x27;;</b>
<b>+┊   ┊ 15┊import { NavigationActions } from &#x27;react-navigation&#x27;;</b>
<b>+┊   ┊ 16┊import update from &#x27;immutability-helper&#x27;;</b>
<b>+┊   ┊ 17┊</b>
<b>+┊   ┊ 18┊import { USER_QUERY } from &#x27;../graphql/user.query&#x27;;</b>
<b>+┊   ┊ 19┊import CREATE_GROUP_MUTATION from &#x27;../graphql/create-group.mutation&#x27;;</b>
<b>+┊   ┊ 20┊import SelectedUserList from &#x27;../components/selected-user-list.component&#x27;;</b>
<b>+┊   ┊ 21┊</b>
<b>+┊   ┊ 22┊const goToNewGroup &#x3D; group &#x3D;&gt; NavigationActions.reset({</b>
<b>+┊   ┊ 23┊  index: 1,</b>
<b>+┊   ┊ 24┊  actions: [</b>
<b>+┊   ┊ 25┊    NavigationActions.navigate({ routeName: &#x27;Main&#x27; }),</b>
<b>+┊   ┊ 26┊    NavigationActions.navigate({ routeName: &#x27;Messages&#x27;, params: { groupId: group.id, title: group.name } }),</b>
<b>+┊   ┊ 27┊  ],</b>
<b>+┊   ┊ 28┊});</b>
<b>+┊   ┊ 29┊</b>
<b>+┊   ┊ 30┊const styles &#x3D; StyleSheet.create({</b>
<b>+┊   ┊ 31┊  container: {</b>
<b>+┊   ┊ 32┊    flex: 1,</b>
<b>+┊   ┊ 33┊    backgroundColor: &#x27;white&#x27;,</b>
<b>+┊   ┊ 34┊  },</b>
<b>+┊   ┊ 35┊  detailsContainer: {</b>
<b>+┊   ┊ 36┊    padding: 20,</b>
<b>+┊   ┊ 37┊    flexDirection: &#x27;row&#x27;,</b>
<b>+┊   ┊ 38┊  },</b>
<b>+┊   ┊ 39┊  imageContainer: {</b>
<b>+┊   ┊ 40┊    paddingRight: 20,</b>
<b>+┊   ┊ 41┊    alignItems: &#x27;center&#x27;,</b>
<b>+┊   ┊ 42┊  },</b>
<b>+┊   ┊ 43┊  inputContainer: {</b>
<b>+┊   ┊ 44┊    flexDirection: &#x27;column&#x27;,</b>
<b>+┊   ┊ 45┊    flex: 1,</b>
<b>+┊   ┊ 46┊  },</b>
<b>+┊   ┊ 47┊  input: {</b>
<b>+┊   ┊ 48┊    color: &#x27;black&#x27;,</b>
<b>+┊   ┊ 49┊    height: 32,</b>
<b>+┊   ┊ 50┊  },</b>
<b>+┊   ┊ 51┊  inputBorder: {</b>
<b>+┊   ┊ 52┊    borderColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊   ┊ 53┊    borderBottomWidth: 1,</b>
<b>+┊   ┊ 54┊    borderTopWidth: 1,</b>
<b>+┊   ┊ 55┊    paddingVertical: 8,</b>
<b>+┊   ┊ 56┊  },</b>
<b>+┊   ┊ 57┊  inputInstructions: {</b>
<b>+┊   ┊ 58┊    paddingTop: 6,</b>
<b>+┊   ┊ 59┊    color: &#x27;#777&#x27;,</b>
<b>+┊   ┊ 60┊    fontSize: 12,</b>
<b>+┊   ┊ 61┊  },</b>
<b>+┊   ┊ 62┊  groupImage: {</b>
<b>+┊   ┊ 63┊    width: 54,</b>
<b>+┊   ┊ 64┊    height: 54,</b>
<b>+┊   ┊ 65┊    borderRadius: 27,</b>
<b>+┊   ┊ 66┊  },</b>
<b>+┊   ┊ 67┊  selected: {</b>
<b>+┊   ┊ 68┊    flexDirection: &#x27;row&#x27;,</b>
<b>+┊   ┊ 69┊  },</b>
<b>+┊   ┊ 70┊  loading: {</b>
<b>+┊   ┊ 71┊    justifyContent: &#x27;center&#x27;,</b>
<b>+┊   ┊ 72┊    flex: 1,</b>
<b>+┊   ┊ 73┊  },</b>
<b>+┊   ┊ 74┊  navIcon: {</b>
<b>+┊   ┊ 75┊    color: &#x27;blue&#x27;,</b>
<b>+┊   ┊ 76┊    fontSize: 18,</b>
<b>+┊   ┊ 77┊    paddingTop: 2,</b>
<b>+┊   ┊ 78┊  },</b>
<b>+┊   ┊ 79┊  participants: {</b>
<b>+┊   ┊ 80┊    paddingHorizontal: 20,</b>
<b>+┊   ┊ 81┊    paddingVertical: 6,</b>
<b>+┊   ┊ 82┊    backgroundColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊   ┊ 83┊    color: &#x27;#777&#x27;,</b>
<b>+┊   ┊ 84┊  },</b>
<b>+┊   ┊ 85┊});</b>
<b>+┊   ┊ 86┊</b>
<b>+┊   ┊ 87┊class FinalizeGroup extends Component {</b>
<b>+┊   ┊ 88┊  static navigationOptions &#x3D; ({ navigation }) &#x3D;&gt; {</b>
<b>+┊   ┊ 89┊    const { state } &#x3D; navigation;</b>
<b>+┊   ┊ 90┊    const isReady &#x3D; state.params &amp;&amp; state.params.mode &#x3D;&#x3D;&#x3D; &#x27;ready&#x27;;</b>
<b>+┊   ┊ 91┊    return {</b>
<b>+┊   ┊ 92┊      title: &#x27;New Group&#x27;,</b>
<b>+┊   ┊ 93┊      headerRight: (</b>
<b>+┊   ┊ 94┊        isReady ? &lt;Button</b>
<b>+┊   ┊ 95┊          title&#x3D;&quot;Create&quot;</b>
<b>+┊   ┊ 96┊          onPress&#x3D;{state.params.create}</b>
<b>+┊   ┊ 97┊        /&gt; : undefined</b>
<b>+┊   ┊ 98┊      ),</b>
<b>+┊   ┊ 99┊    };</b>
<b>+┊   ┊100┊  };</b>
<b>+┊   ┊101┊</b>
<b>+┊   ┊102┊  constructor(props) {</b>
<b>+┊   ┊103┊    super(props);</b>
<b>+┊   ┊104┊</b>
<b>+┊   ┊105┊    const { selected } &#x3D; props.navigation.state.params;</b>
<b>+┊   ┊106┊</b>
<b>+┊   ┊107┊    this.state &#x3D; {</b>
<b>+┊   ┊108┊      selected,</b>
<b>+┊   ┊109┊    };</b>
<b>+┊   ┊110┊</b>
<b>+┊   ┊111┊    this.create &#x3D; this.create.bind(this);</b>
<b>+┊   ┊112┊    this.pop &#x3D; this.pop.bind(this);</b>
<b>+┊   ┊113┊    this.remove &#x3D; this.remove.bind(this);</b>
<b>+┊   ┊114┊  }</b>
<b>+┊   ┊115┊</b>
<b>+┊   ┊116┊  componentDidMount() {</b>
<b>+┊   ┊117┊    this.refreshNavigation(this.state.selected.length &amp;&amp; this.state.name);</b>
<b>+┊   ┊118┊  }</b>
<b>+┊   ┊119┊</b>
<b>+┊   ┊120┊  componentWillUpdate(nextProps, nextState) {</b>
<b>+┊   ┊121┊    if ((nextState.selected.length &amp;&amp; nextState.name) !&#x3D;&#x3D;</b>
<b>+┊   ┊122┊      (this.state.selected.length &amp;&amp; this.state.name)) {</b>
<b>+┊   ┊123┊      this.refreshNavigation(nextState.selected.length &amp;&amp; nextState.name);</b>
<b>+┊   ┊124┊    }</b>
<b>+┊   ┊125┊  }</b>
<b>+┊   ┊126┊</b>
<b>+┊   ┊127┊  pop() {</b>
<b>+┊   ┊128┊    this.props.navigation.goBack();</b>
<b>+┊   ┊129┊  }</b>
<b>+┊   ┊130┊</b>
<b>+┊   ┊131┊  remove(user) {</b>
<b>+┊   ┊132┊    const index &#x3D; this.state.selected.indexOf(user);</b>
<b>+┊   ┊133┊    if (~index) {</b>
<b>+┊   ┊134┊      const selected &#x3D; update(this.state.selected, { $splice: [[index, 1]] });</b>
<b>+┊   ┊135┊      this.setState({</b>
<b>+┊   ┊136┊        selected,</b>
<b>+┊   ┊137┊      });</b>
<b>+┊   ┊138┊    }</b>
<b>+┊   ┊139┊  }</b>
<b>+┊   ┊140┊</b>
<b>+┊   ┊141┊  create() {</b>
<b>+┊   ┊142┊    const { createGroup } &#x3D; this.props;</b>
<b>+┊   ┊143┊</b>
<b>+┊   ┊144┊    createGroup({</b>
<b>+┊   ┊145┊      name: this.state.name,</b>
<b>+┊   ┊146┊      userId: 1, // fake user for now</b>
<b>+┊   ┊147┊      userIds: _.map(this.state.selected, &#x27;id&#x27;),</b>
<b>+┊   ┊148┊    }).then((res) &#x3D;&gt; {</b>
<b>+┊   ┊149┊      this.props.navigation.dispatch(goToNewGroup(res.data.createGroup));</b>
<b>+┊   ┊150┊    }).catch((error) &#x3D;&gt; {</b>
<b>+┊   ┊151┊      Alert.alert(</b>
<b>+┊   ┊152┊        &#x27;Error Creating New Group&#x27;,</b>
<b>+┊   ┊153┊        error.message,</b>
<b>+┊   ┊154┊        [</b>
<b>+┊   ┊155┊          { text: &#x27;OK&#x27;, onPress: () &#x3D;&gt; {} },</b>
<b>+┊   ┊156┊        ],</b>
<b>+┊   ┊157┊      );</b>
<b>+┊   ┊158┊    });</b>
<b>+┊   ┊159┊  }</b>
<b>+┊   ┊160┊</b>
<b>+┊   ┊161┊  refreshNavigation(ready) {</b>
<b>+┊   ┊162┊    const { navigation } &#x3D; this.props;</b>
<b>+┊   ┊163┊    navigation.setParams({</b>
<b>+┊   ┊164┊      mode: ready ? &#x27;ready&#x27; : undefined,</b>
<b>+┊   ┊165┊      create: this.create,</b>
<b>+┊   ┊166┊    });</b>
<b>+┊   ┊167┊  }</b>
<b>+┊   ┊168┊</b>
<b>+┊   ┊169┊  render() {</b>
<b>+┊   ┊170┊    const { friendCount } &#x3D; this.props.navigation.state.params;</b>
<b>+┊   ┊171┊</b>
<b>+┊   ┊172┊    return (</b>
<b>+┊   ┊173┊      &lt;View style&#x3D;{styles.container}&gt;</b>
<b>+┊   ┊174┊        &lt;View style&#x3D;{styles.detailsContainer}&gt;</b>
<b>+┊   ┊175┊          &lt;TouchableOpacity style&#x3D;{styles.imageContainer}&gt;</b>
<b>+┊   ┊176┊            &lt;Image</b>
<b>+┊   ┊177┊              style&#x3D;{styles.groupImage}</b>
<b>+┊   ┊178┊              source&#x3D;{{ uri: &#x27;https://reactjs.org/logo-og.png&#x27; }}</b>
<b>+┊   ┊179┊            /&gt;</b>
<b>+┊   ┊180┊            &lt;Text&gt;edit&lt;/Text&gt;</b>
<b>+┊   ┊181┊          &lt;/TouchableOpacity&gt;</b>
<b>+┊   ┊182┊          &lt;View style&#x3D;{styles.inputContainer}&gt;</b>
<b>+┊   ┊183┊            &lt;View style&#x3D;{styles.inputBorder}&gt;</b>
<b>+┊   ┊184┊              &lt;TextInput</b>
<b>+┊   ┊185┊                autoFocus</b>
<b>+┊   ┊186┊                onChangeText&#x3D;{name &#x3D;&gt; this.setState({ name })}</b>
<b>+┊   ┊187┊                placeholder&#x3D;&quot;Group Subject&quot;</b>
<b>+┊   ┊188┊                style&#x3D;{styles.input}</b>
<b>+┊   ┊189┊              /&gt;</b>
<b>+┊   ┊190┊            &lt;/View&gt;</b>
<b>+┊   ┊191┊            &lt;Text style&#x3D;{styles.inputInstructions}&gt;</b>
<b>+┊   ┊192┊              {&#x27;Please provide a group subject and optional group icon&#x27;}</b>
<b>+┊   ┊193┊            &lt;/Text&gt;</b>
<b>+┊   ┊194┊          &lt;/View&gt;</b>
<b>+┊   ┊195┊        &lt;/View&gt;</b>
<b>+┊   ┊196┊        &lt;Text style&#x3D;{styles.participants}&gt;</b>
<b>+┊   ┊197┊          {&#x60;participants: ${this.state.selected.length} of ${friendCount}&#x60;.toUpperCase()}</b>
<b>+┊   ┊198┊        &lt;/Text&gt;</b>
<b>+┊   ┊199┊        &lt;View style&#x3D;{styles.selected}&gt;</b>
<b>+┊   ┊200┊          {this.state.selected.length ?</b>
<b>+┊   ┊201┊            &lt;SelectedUserList</b>
<b>+┊   ┊202┊              data&#x3D;{this.state.selected}</b>
<b>+┊   ┊203┊              remove&#x3D;{this.remove}</b>
<b>+┊   ┊204┊            /&gt; : undefined}</b>
<b>+┊   ┊205┊        &lt;/View&gt;</b>
<b>+┊   ┊206┊      &lt;/View&gt;</b>
<b>+┊   ┊207┊    );</b>
<b>+┊   ┊208┊  }</b>
<b>+┊   ┊209┊}</b>
<b>+┊   ┊210┊</b>
<b>+┊   ┊211┊FinalizeGroup.propTypes &#x3D; {</b>
<b>+┊   ┊212┊  createGroup: PropTypes.func.isRequired,</b>
<b>+┊   ┊213┊  navigation: PropTypes.shape({</b>
<b>+┊   ┊214┊    dispatch: PropTypes.func,</b>
<b>+┊   ┊215┊    goBack: PropTypes.func,</b>
<b>+┊   ┊216┊    state: PropTypes.shape({</b>
<b>+┊   ┊217┊      params: PropTypes.shape({</b>
<b>+┊   ┊218┊        friendCount: PropTypes.number.isRequired,</b>
<b>+┊   ┊219┊      }),</b>
<b>+┊   ┊220┊    }),</b>
<b>+┊   ┊221┊  }),</b>
<b>+┊   ┊222┊};</b>
<b>+┊   ┊223┊</b>
<b>+┊   ┊224┊const createGroupMutation &#x3D; graphql(CREATE_GROUP_MUTATION, {</b>
<b>+┊   ┊225┊  props: ({ mutate }) &#x3D;&gt; ({</b>
<b>+┊   ┊226┊    createGroup: ({ name, userIds, userId }) &#x3D;&gt;</b>
<b>+┊   ┊227┊      mutate({</b>
<b>+┊   ┊228┊        variables: { name, userIds, userId },</b>
<b>+┊   ┊229┊        update: (store, { data: { createGroup } }) &#x3D;&gt; {</b>
<b>+┊   ┊230┊          // Read the data from our cache for this query.</b>
<b>+┊   ┊231┊          const data &#x3D; store.readQuery({ query: USER_QUERY, variables: { id: userId } });</b>
<b>+┊   ┊232┊</b>
<b>+┊   ┊233┊          // Add our message from the mutation to the end.</b>
<b>+┊   ┊234┊          data.user.groups.push(createGroup);</b>
<b>+┊   ┊235┊</b>
<b>+┊   ┊236┊          // Write our data back to the cache.</b>
<b>+┊   ┊237┊          store.writeQuery({</b>
<b>+┊   ┊238┊            query: USER_QUERY,</b>
<b>+┊   ┊239┊            variables: { id: userId },</b>
<b>+┊   ┊240┊            data,</b>
<b>+┊   ┊241┊          });</b>
<b>+┊   ┊242┊        },</b>
<b>+┊   ┊243┊      }),</b>
<b>+┊   ┊244┊  }),</b>
<b>+┊   ┊245┊});</b>
<b>+┊   ┊246┊</b>
<b>+┊   ┊247┊const userQuery &#x3D; graphql(USER_QUERY, {</b>
<b>+┊   ┊248┊  options: ownProps &#x3D;&gt; ({</b>
<b>+┊   ┊249┊    variables: {</b>
<b>+┊   ┊250┊      id: ownProps.navigation.state.params.userId,</b>
<b>+┊   ┊251┊    },</b>
<b>+┊   ┊252┊  }),</b>
<b>+┊   ┊253┊  props: ({ data: { loading, user } }) &#x3D;&gt; ({</b>
<b>+┊   ┊254┊    loading, user,</b>
<b>+┊   ┊255┊  }),</b>
<b>+┊   ┊256┊});</b>
<b>+┊   ┊257┊</b>
<b>+┊   ┊258┊export default compose(</b>
<b>+┊   ┊259┊  userQuery,</b>
<b>+┊   ┊260┊  createGroupMutation,</b>
<b>+┊   ┊261┊)(FinalizeGroup);</b>
</pre>

##### Added client&#x2F;src&#x2F;screens&#x2F;group-details.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊   ┊  1┊// TODO: update group functionality</b>
<b>+┊   ┊  2┊import React, { Component } from &#x27;react&#x27;;</b>
<b>+┊   ┊  3┊import PropTypes from &#x27;prop-types&#x27;;</b>
<b>+┊   ┊  4┊import {</b>
<b>+┊   ┊  5┊  ActivityIndicator,</b>
<b>+┊   ┊  6┊  Button,</b>
<b>+┊   ┊  7┊  Image,</b>
<b>+┊   ┊  8┊  FlatList,</b>
<b>+┊   ┊  9┊  StyleSheet,</b>
<b>+┊   ┊ 10┊  Text,</b>
<b>+┊   ┊ 11┊  TouchableOpacity,</b>
<b>+┊   ┊ 12┊  View,</b>
<b>+┊   ┊ 13┊} from &#x27;react-native&#x27;;</b>
<b>+┊   ┊ 14┊import { graphql, compose } from &#x27;react-apollo&#x27;;</b>
<b>+┊   ┊ 15┊import { NavigationActions } from &#x27;react-navigation&#x27;;</b>
<b>+┊   ┊ 16┊</b>
<b>+┊   ┊ 17┊import GROUP_QUERY from &#x27;../graphql/group.query&#x27;;</b>
<b>+┊   ┊ 18┊import USER_QUERY from &#x27;../graphql/user.query&#x27;;</b>
<b>+┊   ┊ 19┊import DELETE_GROUP_MUTATION from &#x27;../graphql/delete-group.mutation&#x27;;</b>
<b>+┊   ┊ 20┊import LEAVE_GROUP_MUTATION from &#x27;../graphql/leave-group.mutation&#x27;;</b>
<b>+┊   ┊ 21┊</b>
<b>+┊   ┊ 22┊const resetAction &#x3D; NavigationActions.reset({</b>
<b>+┊   ┊ 23┊  index: 0,</b>
<b>+┊   ┊ 24┊  actions: [</b>
<b>+┊   ┊ 25┊    NavigationActions.navigate({ routeName: &#x27;Main&#x27; }),</b>
<b>+┊   ┊ 26┊  ],</b>
<b>+┊   ┊ 27┊});</b>
<b>+┊   ┊ 28┊</b>
<b>+┊   ┊ 29┊const styles &#x3D; StyleSheet.create({</b>
<b>+┊   ┊ 30┊  container: {</b>
<b>+┊   ┊ 31┊    flex: 1,</b>
<b>+┊   ┊ 32┊  },</b>
<b>+┊   ┊ 33┊  avatar: {</b>
<b>+┊   ┊ 34┊    width: 32,</b>
<b>+┊   ┊ 35┊    height: 32,</b>
<b>+┊   ┊ 36┊    borderRadius: 16,</b>
<b>+┊   ┊ 37┊  },</b>
<b>+┊   ┊ 38┊  detailsContainer: {</b>
<b>+┊   ┊ 39┊    flexDirection: &#x27;row&#x27;,</b>
<b>+┊   ┊ 40┊    alignItems: &#x27;center&#x27;,</b>
<b>+┊   ┊ 41┊  },</b>
<b>+┊   ┊ 42┊  groupImageContainer: {</b>
<b>+┊   ┊ 43┊    paddingTop: 20,</b>
<b>+┊   ┊ 44┊    paddingHorizontal: 20,</b>
<b>+┊   ┊ 45┊    paddingBottom: 6,</b>
<b>+┊   ┊ 46┊    alignItems: &#x27;center&#x27;,</b>
<b>+┊   ┊ 47┊  },</b>
<b>+┊   ┊ 48┊  groupName: {</b>
<b>+┊   ┊ 49┊    color: &#x27;black&#x27;,</b>
<b>+┊   ┊ 50┊  },</b>
<b>+┊   ┊ 51┊  groupNameBorder: {</b>
<b>+┊   ┊ 52┊    borderBottomWidth: 1,</b>
<b>+┊   ┊ 53┊    borderColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊   ┊ 54┊    borderTopWidth: 1,</b>
<b>+┊   ┊ 55┊    flex: 1,</b>
<b>+┊   ┊ 56┊    paddingVertical: 8,</b>
<b>+┊   ┊ 57┊  },</b>
<b>+┊   ┊ 58┊  groupImage: {</b>
<b>+┊   ┊ 59┊    width: 54,</b>
<b>+┊   ┊ 60┊    height: 54,</b>
<b>+┊   ┊ 61┊    borderRadius: 27,</b>
<b>+┊   ┊ 62┊  },</b>
<b>+┊   ┊ 63┊  participants: {</b>
<b>+┊   ┊ 64┊    borderBottomWidth: 1,</b>
<b>+┊   ┊ 65┊    borderColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊   ┊ 66┊    borderTopWidth: 1,</b>
<b>+┊   ┊ 67┊    paddingHorizontal: 20,</b>
<b>+┊   ┊ 68┊    paddingVertical: 6,</b>
<b>+┊   ┊ 69┊    backgroundColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊   ┊ 70┊    color: &#x27;#777&#x27;,</b>
<b>+┊   ┊ 71┊  },</b>
<b>+┊   ┊ 72┊  user: {</b>
<b>+┊   ┊ 73┊    alignItems: &#x27;center&#x27;,</b>
<b>+┊   ┊ 74┊    borderBottomWidth: 1,</b>
<b>+┊   ┊ 75┊    borderColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊   ┊ 76┊    flexDirection: &#x27;row&#x27;,</b>
<b>+┊   ┊ 77┊    padding: 10,</b>
<b>+┊   ┊ 78┊  },</b>
<b>+┊   ┊ 79┊  username: {</b>
<b>+┊   ┊ 80┊    flex: 1,</b>
<b>+┊   ┊ 81┊    fontSize: 16,</b>
<b>+┊   ┊ 82┊    paddingHorizontal: 12,</b>
<b>+┊   ┊ 83┊    paddingVertical: 8,</b>
<b>+┊   ┊ 84┊  },</b>
<b>+┊   ┊ 85┊});</b>
<b>+┊   ┊ 86┊</b>
<b>+┊   ┊ 87┊class GroupDetails extends Component {</b>
<b>+┊   ┊ 88┊  static navigationOptions &#x3D; ({ navigation }) &#x3D;&gt; ({</b>
<b>+┊   ┊ 89┊    title: &#x60;${navigation.state.params.title}&#x60;,</b>
<b>+┊   ┊ 90┊  });</b>
<b>+┊   ┊ 91┊</b>
<b>+┊   ┊ 92┊  constructor(props) {</b>
<b>+┊   ┊ 93┊    super(props);</b>
<b>+┊   ┊ 94┊</b>
<b>+┊   ┊ 95┊    this.deleteGroup &#x3D; this.deleteGroup.bind(this);</b>
<b>+┊   ┊ 96┊    this.leaveGroup &#x3D; this.leaveGroup.bind(this);</b>
<b>+┊   ┊ 97┊    this.renderItem &#x3D; this.renderItem.bind(this);</b>
<b>+┊   ┊ 98┊  }</b>
<b>+┊   ┊ 99┊</b>
<b>+┊   ┊100┊  deleteGroup() {</b>
<b>+┊   ┊101┊    this.props.deleteGroup(this.props.navigation.state.params.id)</b>
<b>+┊   ┊102┊      .then(() &#x3D;&gt; {</b>
<b>+┊   ┊103┊        this.props.navigation.dispatch(resetAction);</b>
<b>+┊   ┊104┊      })</b>
<b>+┊   ┊105┊      .catch((e) &#x3D;&gt; {</b>
<b>+┊   ┊106┊        console.log(e); // eslint-disable-line no-console</b>
<b>+┊   ┊107┊      });</b>
<b>+┊   ┊108┊  }</b>
<b>+┊   ┊109┊</b>
<b>+┊   ┊110┊  leaveGroup() {</b>
<b>+┊   ┊111┊    this.props.leaveGroup({</b>
<b>+┊   ┊112┊      id: this.props.navigation.state.params.id,</b>
<b>+┊   ┊113┊      userId: 1,</b>
<b>+┊   ┊114┊    }) // fake user for now</b>
<b>+┊   ┊115┊      .then(() &#x3D;&gt; {</b>
<b>+┊   ┊116┊        this.props.navigation.dispatch(resetAction);</b>
<b>+┊   ┊117┊      })</b>
<b>+┊   ┊118┊      .catch((e) &#x3D;&gt; {</b>
<b>+┊   ┊119┊        console.log(e); // eslint-disable-line no-console</b>
<b>+┊   ┊120┊      });</b>
<b>+┊   ┊121┊  }</b>
<b>+┊   ┊122┊</b>
<b>+┊   ┊123┊  keyExtractor &#x3D; item &#x3D;&gt; item.id.toString();</b>
<b>+┊   ┊124┊</b>
<b>+┊   ┊125┊  renderItem &#x3D; ({ item: user }) &#x3D;&gt; (</b>
<b>+┊   ┊126┊    &lt;View style&#x3D;{styles.user}&gt;</b>
<b>+┊   ┊127┊      &lt;Image</b>
<b>+┊   ┊128┊        style&#x3D;{styles.avatar}</b>
<b>+┊   ┊129┊        source&#x3D;{{ uri: &#x27;https://reactjs.org/logo-og.png&#x27; }}</b>
<b>+┊   ┊130┊      /&gt;</b>
<b>+┊   ┊131┊      &lt;Text style&#x3D;{styles.username}&gt;{user.username}&lt;/Text&gt;</b>
<b>+┊   ┊132┊    &lt;/View&gt;</b>
<b>+┊   ┊133┊  )</b>
<b>+┊   ┊134┊</b>
<b>+┊   ┊135┊  render() {</b>
<b>+┊   ┊136┊    const { group, loading } &#x3D; this.props;</b>
<b>+┊   ┊137┊</b>
<b>+┊   ┊138┊    // render loading placeholder while we fetch messages</b>
<b>+┊   ┊139┊    if (!group || loading) {</b>
<b>+┊   ┊140┊      return (</b>
<b>+┊   ┊141┊        &lt;View style&#x3D;{[styles.loading, styles.container]}&gt;</b>
<b>+┊   ┊142┊          &lt;ActivityIndicator /&gt;</b>
<b>+┊   ┊143┊        &lt;/View&gt;</b>
<b>+┊   ┊144┊      );</b>
<b>+┊   ┊145┊    }</b>
<b>+┊   ┊146┊</b>
<b>+┊   ┊147┊    return (</b>
<b>+┊   ┊148┊      &lt;View style&#x3D;{styles.container}&gt;</b>
<b>+┊   ┊149┊        &lt;FlatList</b>
<b>+┊   ┊150┊          data&#x3D;{group.users}</b>
<b>+┊   ┊151┊          keyExtractor&#x3D;{this.keyExtractor}</b>
<b>+┊   ┊152┊          renderItem&#x3D;{this.renderItem}</b>
<b>+┊   ┊153┊          ListHeaderComponent&#x3D;{() &#x3D;&gt; (</b>
<b>+┊   ┊154┊            &lt;View&gt;</b>
<b>+┊   ┊155┊              &lt;View style&#x3D;{styles.detailsContainer}&gt;</b>
<b>+┊   ┊156┊                &lt;TouchableOpacity style&#x3D;{styles.groupImageContainer} onPress&#x3D;{this.pickGroupImage}&gt;</b>
<b>+┊   ┊157┊                  &lt;Image</b>
<b>+┊   ┊158┊                    style&#x3D;{styles.groupImage}</b>
<b>+┊   ┊159┊                    source&#x3D;{{ uri: &#x27;https://reactjs.org/logo-og.png&#x27; }}</b>
<b>+┊   ┊160┊                  /&gt;</b>
<b>+┊   ┊161┊                  &lt;Text&gt;edit&lt;/Text&gt;</b>
<b>+┊   ┊162┊                &lt;/TouchableOpacity&gt;</b>
<b>+┊   ┊163┊                &lt;View style&#x3D;{styles.groupNameBorder}&gt;</b>
<b>+┊   ┊164┊                  &lt;Text style&#x3D;{styles.groupName}&gt;{group.name}&lt;/Text&gt;</b>
<b>+┊   ┊165┊                &lt;/View&gt;</b>
<b>+┊   ┊166┊              &lt;/View&gt;</b>
<b>+┊   ┊167┊              &lt;Text style&#x3D;{styles.participants}&gt;</b>
<b>+┊   ┊168┊                {&#x60;participants: ${group.users.length}&#x60;.toUpperCase()}</b>
<b>+┊   ┊169┊              &lt;/Text&gt;</b>
<b>+┊   ┊170┊            &lt;/View&gt;</b>
<b>+┊   ┊171┊          )}</b>
<b>+┊   ┊172┊          ListFooterComponent&#x3D;{() &#x3D;&gt; (</b>
<b>+┊   ┊173┊            &lt;View&gt;</b>
<b>+┊   ┊174┊              &lt;Button title&#x3D;&quot;Leave Group&quot; onPress&#x3D;{this.leaveGroup} /&gt;</b>
<b>+┊   ┊175┊              &lt;Button title&#x3D;&quot;Delete Group&quot; onPress&#x3D;{this.deleteGroup} /&gt;</b>
<b>+┊   ┊176┊            &lt;/View&gt;</b>
<b>+┊   ┊177┊          )}</b>
<b>+┊   ┊178┊        /&gt;</b>
<b>+┊   ┊179┊      &lt;/View&gt;</b>
<b>+┊   ┊180┊    );</b>
<b>+┊   ┊181┊  }</b>
<b>+┊   ┊182┊}</b>
<b>+┊   ┊183┊</b>
<b>+┊   ┊184┊GroupDetails.propTypes &#x3D; {</b>
<b>+┊   ┊185┊  loading: PropTypes.bool,</b>
<b>+┊   ┊186┊  group: PropTypes.shape({</b>
<b>+┊   ┊187┊    id: PropTypes.number,</b>
<b>+┊   ┊188┊    name: PropTypes.string,</b>
<b>+┊   ┊189┊    users: PropTypes.arrayOf(PropTypes.shape({</b>
<b>+┊   ┊190┊      id: PropTypes.number,</b>
<b>+┊   ┊191┊      username: PropTypes.string,</b>
<b>+┊   ┊192┊    })),</b>
<b>+┊   ┊193┊  }),</b>
<b>+┊   ┊194┊  navigation: PropTypes.shape({</b>
<b>+┊   ┊195┊    dispatch: PropTypes.func,</b>
<b>+┊   ┊196┊    state: PropTypes.shape({</b>
<b>+┊   ┊197┊      params: PropTypes.shape({</b>
<b>+┊   ┊198┊        title: PropTypes.string,</b>
<b>+┊   ┊199┊        id: PropTypes.number,</b>
<b>+┊   ┊200┊      }),</b>
<b>+┊   ┊201┊    }),</b>
<b>+┊   ┊202┊  }),</b>
<b>+┊   ┊203┊  deleteGroup: PropTypes.func.isRequired,</b>
<b>+┊   ┊204┊  leaveGroup: PropTypes.func.isRequired,</b>
<b>+┊   ┊205┊};</b>
<b>+┊   ┊206┊</b>
<b>+┊   ┊207┊const groupQuery &#x3D; graphql(GROUP_QUERY, {</b>
<b>+┊   ┊208┊  options: ownProps &#x3D;&gt; ({ variables: { groupId: ownProps.navigation.state.params.id } }),</b>
<b>+┊   ┊209┊  props: ({ data: { loading, group } }) &#x3D;&gt; ({</b>
<b>+┊   ┊210┊    loading,</b>
<b>+┊   ┊211┊    group,</b>
<b>+┊   ┊212┊  }),</b>
<b>+┊   ┊213┊});</b>
<b>+┊   ┊214┊</b>
<b>+┊   ┊215┊const deleteGroupMutation &#x3D; graphql(DELETE_GROUP_MUTATION, {</b>
<b>+┊   ┊216┊  props: ({ ownProps, mutate }) &#x3D;&gt; ({</b>
<b>+┊   ┊217┊    deleteGroup: id &#x3D;&gt;</b>
<b>+┊   ┊218┊      mutate({</b>
<b>+┊   ┊219┊        variables: { id },</b>
<b>+┊   ┊220┊        update: (store, { data: { deleteGroup } }) &#x3D;&gt; {</b>
<b>+┊   ┊221┊          // Read the data from our cache for this query.</b>
<b>+┊   ┊222┊          const data &#x3D; store.readQuery({ query: USER_QUERY, variables: { id: 1 } }); // fake for now</b>
<b>+┊   ┊223┊</b>
<b>+┊   ┊224┊          // Add our message from the mutation to the end.</b>
<b>+┊   ┊225┊          data.user.groups &#x3D; data.user.groups.filter(g &#x3D;&gt; deleteGroup.id !&#x3D;&#x3D; g.id);</b>
<b>+┊   ┊226┊</b>
<b>+┊   ┊227┊          // Write our data back to the cache.</b>
<b>+┊   ┊228┊          store.writeQuery({</b>
<b>+┊   ┊229┊            query: USER_QUERY,</b>
<b>+┊   ┊230┊            variables: { id: 1 }, // fake for now</b>
<b>+┊   ┊231┊            data,</b>
<b>+┊   ┊232┊          });</b>
<b>+┊   ┊233┊        },</b>
<b>+┊   ┊234┊      }),</b>
<b>+┊   ┊235┊  }),</b>
<b>+┊   ┊236┊});</b>
<b>+┊   ┊237┊</b>
<b>+┊   ┊238┊const leaveGroupMutation &#x3D; graphql(LEAVE_GROUP_MUTATION, {</b>
<b>+┊   ┊239┊  props: ({ ownProps, mutate }) &#x3D;&gt; ({</b>
<b>+┊   ┊240┊    leaveGroup: ({ id, userId }) &#x3D;&gt;</b>
<b>+┊   ┊241┊      mutate({</b>
<b>+┊   ┊242┊        variables: { id, userId },</b>
<b>+┊   ┊243┊        update: (store, { data: { leaveGroup } }) &#x3D;&gt; {</b>
<b>+┊   ┊244┊          // Read the data from our cache for this query.</b>
<b>+┊   ┊245┊          const data &#x3D; store.readQuery({ query: USER_QUERY, variables: { id: 1 } }); // fake for now</b>
<b>+┊   ┊246┊</b>
<b>+┊   ┊247┊          // Add our message from the mutation to the end.</b>
<b>+┊   ┊248┊          data.user.groups &#x3D; data.user.groups.filter(g &#x3D;&gt; leaveGroup.id !&#x3D;&#x3D; g.id);</b>
<b>+┊   ┊249┊</b>
<b>+┊   ┊250┊          // Write our data back to the cache.</b>
<b>+┊   ┊251┊          store.writeQuery({</b>
<b>+┊   ┊252┊            query: USER_QUERY,</b>
<b>+┊   ┊253┊            variables: { id: 1 }, // fake for now</b>
<b>+┊   ┊254┊            data,</b>
<b>+┊   ┊255┊          });</b>
<b>+┊   ┊256┊        },</b>
<b>+┊   ┊257┊      }),</b>
<b>+┊   ┊258┊  }),</b>
<b>+┊   ┊259┊});</b>
<b>+┊   ┊260┊</b>
<b>+┊   ┊261┊export default compose(</b>
<b>+┊   ┊262┊  groupQuery,</b>
<b>+┊   ┊263┊  deleteGroupMutation,</b>
<b>+┊   ┊264┊  leaveGroupMutation,</b>
<b>+┊   ┊265┊)(GroupDetails);</b>
</pre>

##### Changed client&#x2F;src&#x2F;screens&#x2F;groups.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊3┊3┊import {
 ┊4┊4┊  FlatList,
 ┊5┊5┊  ActivityIndicator,
<b>+┊ ┊6┊  Button,</b>
 ┊6┊7┊  StyleSheet,
 ┊7┊8┊  Text,
 ┊8┊9┊  TouchableHighlight,
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊35┊36┊    fontWeight: &#x27;bold&#x27;,
 ┊36┊37┊    flex: 0.7,
 ┊37┊38┊  },
<b>+┊  ┊39┊  header: {</b>
<b>+┊  ┊40┊    alignItems: &#x27;flex-end&#x27;,</b>
<b>+┊  ┊41┊    padding: 6,</b>
<b>+┊  ┊42┊    borderColor: &#x27;#eee&#x27;,</b>
<b>+┊  ┊43┊    borderBottomWidth: 1,</b>
<b>+┊  ┊44┊  },</b>
<b>+┊  ┊45┊  warning: {</b>
<b>+┊  ┊46┊    textAlign: &#x27;center&#x27;,</b>
<b>+┊  ┊47┊    padding: 12,</b>
<b>+┊  ┊48┊  },</b>
 ┊38┊49┊});
 ┊39┊50┊
<b>+┊  ┊51┊const Header &#x3D; ({ onPress }) &#x3D;&gt; (</b>
<b>+┊  ┊52┊  &lt;View style&#x3D;{styles.header}&gt;</b>
<b>+┊  ┊53┊    &lt;Button title&#x3D;{&#x27;New Group&#x27;} onPress&#x3D;{onPress} /&gt;</b>
<b>+┊  ┊54┊  &lt;/View&gt;</b>
<b>+┊  ┊55┊);</b>
<b>+┊  ┊56┊Header.propTypes &#x3D; {</b>
<b>+┊  ┊57┊  onPress: PropTypes.func.isRequired,</b>
<b>+┊  ┊58┊};</b>
<b>+┊  ┊59┊</b>
 ┊40┊60┊class Group extends Component {
 ┊41┊61┊  constructor(props) {
 ┊42┊62┊    super(props);
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊ 75┊ 95┊  constructor(props) {
 ┊ 76┊ 96┊    super(props);
 ┊ 77┊ 97┊    this.goToMessages &#x3D; this.goToMessages.bind(this);
<b>+┊   ┊ 98┊    this.goToNewGroup &#x3D; this.goToNewGroup.bind(this);</b>
 ┊ 78┊ 99┊  }
 ┊ 79┊100┊
 ┊ 80┊101┊  keyExtractor &#x3D; item &#x3D;&gt; item.id.toString();
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊ 84┊105┊    navigate(&#x27;Messages&#x27;, { groupId: group.id, title: group.name });
 ┊ 85┊106┊  }
 ┊ 86┊107┊
<b>+┊   ┊108┊  goToNewGroup() {</b>
<b>+┊   ┊109┊    const { navigate } &#x3D; this.props.navigation;</b>
<b>+┊   ┊110┊    navigate(&#x27;NewGroup&#x27;);</b>
<b>+┊   ┊111┊  }</b>
<b>+┊   ┊112┊</b>
 ┊ 87┊113┊  renderItem &#x3D; ({ item }) &#x3D;&gt; &lt;Group group&#x3D;{item} goToMessages&#x3D;{this.goToMessages} /&gt;;
 ┊ 88┊114┊
 ┊ 89┊115┊  render() {
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊ 98┊124┊      );
 ┊ 99┊125┊    }
 ┊100┊126┊
<b>+┊   ┊127┊    if (user &amp;&amp; !user.groups.length) {</b>
<b>+┊   ┊128┊      return (</b>
<b>+┊   ┊129┊        &lt;View style&#x3D;{styles.container}&gt;</b>
<b>+┊   ┊130┊          &lt;Header onPress&#x3D;{this.goToNewGroup} /&gt;</b>
<b>+┊   ┊131┊          &lt;Text style&#x3D;{styles.warning}&gt;{&#x27;You do not have any groups.&#x27;}&lt;/Text&gt;</b>
<b>+┊   ┊132┊        &lt;/View&gt;</b>
<b>+┊   ┊133┊      );</b>
<b>+┊   ┊134┊    }</b>
<b>+┊   ┊135┊</b>
 ┊101┊136┊    // render list of groups for user
 ┊102┊137┊    return (
 ┊103┊138┊      &lt;View style&#x3D;{styles.container}&gt;
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊105┊140┊          data&#x3D;{user.groups}
 ┊106┊141┊          keyExtractor&#x3D;{this.keyExtractor}
 ┊107┊142┊          renderItem&#x3D;{this.renderItem}
<b>+┊   ┊143┊          ListHeaderComponent&#x3D;{() &#x3D;&gt; &lt;Header onPress&#x3D;{this.goToNewGroup} /&gt;}</b>
 ┊108┊144┊        /&gt;
 ┊109┊145┊      &lt;/View&gt;
 ┊110┊146┊    );
</pre>

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊ 1┊ 1┊import {
 ┊ 2┊ 2┊  ActivityIndicator,
 ┊ 3┊ 3┊  FlatList,
<b>+┊  ┊ 4┊  Image,</b>
 ┊ 4┊ 5┊  KeyboardAvoidingView,
 ┊ 5┊ 6┊  StyleSheet,
<b>+┊  ┊ 7┊  Text,</b>
<b>+┊  ┊ 8┊  TouchableOpacity,</b>
 ┊ 6┊ 9┊  View,
 ┊ 7┊10┊} from &#x27;react-native&#x27;;
 ┊ 8┊11┊import PropTypes from &#x27;prop-types&#x27;;
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊25┊28┊  loading: {
 ┊26┊29┊    justifyContent: &#x27;center&#x27;,
 ┊27┊30┊  },
<b>+┊  ┊31┊  titleWrapper: {</b>
<b>+┊  ┊32┊    alignItems: &#x27;center&#x27;,</b>
<b>+┊  ┊33┊    position: &#x27;absolute&#x27;,</b>
<b>+┊  ┊34┊    left: 0,</b>
<b>+┊  ┊35┊    right: 0,</b>
<b>+┊  ┊36┊  },</b>
<b>+┊  ┊37┊  title: {</b>
<b>+┊  ┊38┊    flexDirection: &#x27;row&#x27;,</b>
<b>+┊  ┊39┊    alignItems: &#x27;center&#x27;,</b>
<b>+┊  ┊40┊  },</b>
<b>+┊  ┊41┊  titleImage: {</b>
<b>+┊  ┊42┊    marginRight: 6,</b>
<b>+┊  ┊43┊    width: 32,</b>
<b>+┊  ┊44┊    height: 32,</b>
<b>+┊  ┊45┊    borderRadius: 16,</b>
<b>+┊  ┊46┊  },</b>
 ┊28┊47┊});
 ┊29┊48┊
 ┊30┊49┊class Messages extends Component {
 ┊31┊50┊  static navigationOptions &#x3D; ({ navigation }) &#x3D;&gt; {
<b>+┊  ┊51┊    const { state, navigate } &#x3D; navigation;</b>
<b>+┊  ┊52┊</b>
<b>+┊  ┊53┊    const goToGroupDetails &#x3D; navigate.bind(this, &#x27;GroupDetails&#x27;, {</b>
<b>+┊  ┊54┊      id: state.params.groupId,</b>
 ┊34┊55┊      title: state.params.title,
<b>+┊  ┊56┊    });</b>
<b>+┊  ┊57┊</b>
<b>+┊  ┊58┊    return {</b>
<b>+┊  ┊59┊      headerTitle: (</b>
<b>+┊  ┊60┊        &lt;TouchableOpacity</b>
<b>+┊  ┊61┊          style&#x3D;{styles.titleWrapper}</b>
<b>+┊  ┊62┊          onPress&#x3D;{goToGroupDetails}</b>
<b>+┊  ┊63┊        &gt;</b>
<b>+┊  ┊64┊          &lt;View style&#x3D;{styles.title}&gt;</b>
<b>+┊  ┊65┊            &lt;Image</b>
<b>+┊  ┊66┊              style&#x3D;{styles.titleImage}</b>
<b>+┊  ┊67┊              source&#x3D;{{ uri: &#x27;https://reactjs.org/logo-og.png&#x27; }}</b>
<b>+┊  ┊68┊            /&gt;</b>
<b>+┊  ┊69┊            &lt;Text&gt;{state.params.title}&lt;/Text&gt;</b>
<b>+┊  ┊70┊          &lt;/View&gt;</b>
<b>+┊  ┊71┊        &lt;/TouchableOpacity&gt;</b>
<b>+┊  ┊72┊      ),</b>
 ┊35┊73┊    };
 ┊36┊74┊  };
 ┊37┊75┊
</pre>
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊125┊163┊Messages.propTypes &#x3D; {
 ┊126┊164┊  createMessage: PropTypes.func,
 ┊127┊165┊  navigation: PropTypes.shape({
<b>+┊   ┊166┊    navigate: PropTypes.func,</b>
 ┊128┊167┊    state: PropTypes.shape({
 ┊129┊168┊      params: PropTypes.shape({
 ┊130┊169┊        groupId: PropTypes.number,
</pre>

##### Added client&#x2F;src&#x2F;screens&#x2F;new-group.screen.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
<b>+┊   ┊  1┊import { _ } from &#x27;lodash&#x27;;</b>
<b>+┊   ┊  2┊import React, { Component } from &#x27;react&#x27;;</b>
<b>+┊   ┊  3┊import PropTypes from &#x27;prop-types&#x27;;</b>
<b>+┊   ┊  4┊import {</b>
<b>+┊   ┊  5┊  ActivityIndicator,</b>
<b>+┊   ┊  6┊  Button,</b>
<b>+┊   ┊  7┊  Image,</b>
<b>+┊   ┊  8┊  StyleSheet,</b>
<b>+┊   ┊  9┊  Text,</b>
<b>+┊   ┊ 10┊  View,</b>
<b>+┊   ┊ 11┊} from &#x27;react-native&#x27;;</b>
<b>+┊   ┊ 12┊import { graphql, compose } from &#x27;react-apollo&#x27;;</b>
<b>+┊   ┊ 13┊import AlphabetListView from &#x27;react-native-alpha-listview&#x27;;</b>
<b>+┊   ┊ 14┊import update from &#x27;immutability-helper&#x27;;</b>
<b>+┊   ┊ 15┊import Icon from &#x27;react-native-vector-icons/FontAwesome&#x27;;</b>
<b>+┊   ┊ 16┊</b>
<b>+┊   ┊ 17┊import SelectedUserList from &#x27;../components/selected-user-list.component&#x27;;</b>
<b>+┊   ┊ 18┊import USER_QUERY from &#x27;../graphql/user.query&#x27;;</b>
<b>+┊   ┊ 19┊</b>
<b>+┊   ┊ 20┊// eslint-disable-next-line</b>
<b>+┊   ┊ 21┊const sortObject &#x3D; o &#x3D;&gt; Object.keys(o).sort().reduce((r, k) &#x3D;&gt; (r[k] &#x3D; o[k], r), {});</b>
<b>+┊   ┊ 22┊</b>
<b>+┊   ┊ 23┊const styles &#x3D; StyleSheet.create({</b>
<b>+┊   ┊ 24┊  container: {</b>
<b>+┊   ┊ 25┊    flex: 1,</b>
<b>+┊   ┊ 26┊    backgroundColor: &#x27;white&#x27;,</b>
<b>+┊   ┊ 27┊  },</b>
<b>+┊   ┊ 28┊  cellContainer: {</b>
<b>+┊   ┊ 29┊    alignItems: &#x27;center&#x27;,</b>
<b>+┊   ┊ 30┊    flex: 1,</b>
<b>+┊   ┊ 31┊    flexDirection: &#x27;row&#x27;,</b>
<b>+┊   ┊ 32┊    flexWrap: &#x27;wrap&#x27;,</b>
<b>+┊   ┊ 33┊    paddingHorizontal: 12,</b>
<b>+┊   ┊ 34┊    paddingVertical: 6,</b>
<b>+┊   ┊ 35┊  },</b>
<b>+┊   ┊ 36┊  cellImage: {</b>
<b>+┊   ┊ 37┊    width: 32,</b>
<b>+┊   ┊ 38┊    height: 32,</b>
<b>+┊   ┊ 39┊    borderRadius: 16,</b>
<b>+┊   ┊ 40┊  },</b>
<b>+┊   ┊ 41┊  cellLabel: {</b>
<b>+┊   ┊ 42┊    flex: 1,</b>
<b>+┊   ┊ 43┊    fontSize: 16,</b>
<b>+┊   ┊ 44┊    paddingHorizontal: 12,</b>
<b>+┊   ┊ 45┊    paddingVertical: 8,</b>
<b>+┊   ┊ 46┊  },</b>
<b>+┊   ┊ 47┊  selected: {</b>
<b>+┊   ┊ 48┊    flexDirection: &#x27;row&#x27;,</b>
<b>+┊   ┊ 49┊  },</b>
<b>+┊   ┊ 50┊  loading: {</b>
<b>+┊   ┊ 51┊    justifyContent: &#x27;center&#x27;,</b>
<b>+┊   ┊ 52┊    flex: 1,</b>
<b>+┊   ┊ 53┊  },</b>
<b>+┊   ┊ 54┊  navIcon: {</b>
<b>+┊   ┊ 55┊    color: &#x27;blue&#x27;,</b>
<b>+┊   ┊ 56┊    fontSize: 18,</b>
<b>+┊   ┊ 57┊    paddingTop: 2,</b>
<b>+┊   ┊ 58┊  },</b>
<b>+┊   ┊ 59┊  checkButtonContainer: {</b>
<b>+┊   ┊ 60┊    paddingRight: 12,</b>
<b>+┊   ┊ 61┊    paddingVertical: 6,</b>
<b>+┊   ┊ 62┊  },</b>
<b>+┊   ┊ 63┊  checkButton: {</b>
<b>+┊   ┊ 64┊    borderWidth: 1,</b>
<b>+┊   ┊ 65┊    borderColor: &#x27;#dbdbdb&#x27;,</b>
<b>+┊   ┊ 66┊    padding: 4,</b>
<b>+┊   ┊ 67┊    height: 24,</b>
<b>+┊   ┊ 68┊    width: 24,</b>
<b>+┊   ┊ 69┊  },</b>
<b>+┊   ┊ 70┊  checkButtonIcon: {</b>
<b>+┊   ┊ 71┊    marginRight: -4, // default is 12</b>
<b>+┊   ┊ 72┊  },</b>
<b>+┊   ┊ 73┊});</b>
<b>+┊   ┊ 74┊</b>
<b>+┊   ┊ 75┊const SectionHeader &#x3D; ({ title }) &#x3D;&gt; {</b>
<b>+┊   ┊ 76┊  // inline styles used for brevity, use a stylesheet when possible</b>
<b>+┊   ┊ 77┊  const textStyle &#x3D; {</b>
<b>+┊   ┊ 78┊    textAlign: &#x27;center&#x27;,</b>
<b>+┊   ┊ 79┊    color: &#x27;#fff&#x27;,</b>
<b>+┊   ┊ 80┊    fontWeight: &#x27;700&#x27;,</b>
<b>+┊   ┊ 81┊    fontSize: 16,</b>
<b>+┊   ┊ 82┊  };</b>
<b>+┊   ┊ 83┊</b>
<b>+┊   ┊ 84┊  const viewStyle &#x3D; {</b>
<b>+┊   ┊ 85┊    backgroundColor: &#x27;#ccc&#x27;,</b>
<b>+┊   ┊ 86┊  };</b>
<b>+┊   ┊ 87┊  return (</b>
<b>+┊   ┊ 88┊    &lt;View style&#x3D;{viewStyle}&gt;</b>
<b>+┊   ┊ 89┊      &lt;Text style&#x3D;{textStyle}&gt;{title}&lt;/Text&gt;</b>
<b>+┊   ┊ 90┊    &lt;/View&gt;</b>
<b>+┊   ┊ 91┊  );</b>
<b>+┊   ┊ 92┊};</b>
<b>+┊   ┊ 93┊SectionHeader.propTypes &#x3D; {</b>
<b>+┊   ┊ 94┊  title: PropTypes.string,</b>
<b>+┊   ┊ 95┊};</b>
<b>+┊   ┊ 96┊</b>
<b>+┊   ┊ 97┊const SectionItem &#x3D; ({ title }) &#x3D;&gt; (</b>
<b>+┊   ┊ 98┊  &lt;Text style&#x3D;{{ color: &#x27;blue&#x27; }}&gt;{title}&lt;/Text&gt;</b>
<b>+┊   ┊ 99┊);</b>
<b>+┊   ┊100┊SectionItem.propTypes &#x3D; {</b>
<b>+┊   ┊101┊  title: PropTypes.string,</b>
<b>+┊   ┊102┊};</b>
<b>+┊   ┊103┊</b>
<b>+┊   ┊104┊class Cell extends Component {</b>
<b>+┊   ┊105┊  constructor(props) {</b>
<b>+┊   ┊106┊    super(props);</b>
<b>+┊   ┊107┊    this.toggle &#x3D; this.toggle.bind(this);</b>
<b>+┊   ┊108┊    this.state &#x3D; {</b>
<b>+┊   ┊109┊      isSelected: props.isSelected(props.item),</b>
<b>+┊   ┊110┊    };</b>
<b>+┊   ┊111┊  }</b>
<b>+┊   ┊112┊</b>
<b>+┊   ┊113┊  componentWillReceiveProps(nextProps) {</b>
<b>+┊   ┊114┊    this.setState({</b>
<b>+┊   ┊115┊      isSelected: nextProps.isSelected(nextProps.item),</b>
<b>+┊   ┊116┊    });</b>
<b>+┊   ┊117┊  }</b>
<b>+┊   ┊118┊</b>
<b>+┊   ┊119┊  toggle() {</b>
<b>+┊   ┊120┊    this.props.toggle(this.props.item);</b>
<b>+┊   ┊121┊  }</b>
<b>+┊   ┊122┊</b>
<b>+┊   ┊123┊  render() {</b>
<b>+┊   ┊124┊    return (</b>
<b>+┊   ┊125┊      &lt;View style&#x3D;{styles.cellContainer}&gt;</b>
<b>+┊   ┊126┊        &lt;Image</b>
<b>+┊   ┊127┊          style&#x3D;{styles.cellImage}</b>
<b>+┊   ┊128┊          source&#x3D;{{ uri: &#x27;https://reactjs.org/logo-og.png&#x27; }}</b>
<b>+┊   ┊129┊        /&gt;</b>
<b>+┊   ┊130┊        &lt;Text style&#x3D;{styles.cellLabel}&gt;{this.props.item.username}&lt;/Text&gt;</b>
<b>+┊   ┊131┊        &lt;View style&#x3D;{styles.checkButtonContainer}&gt;</b>
<b>+┊   ┊132┊          &lt;Icon.Button</b>
<b>+┊   ┊133┊            backgroundColor&#x3D;{this.state.isSelected ? &#x27;blue&#x27; : &#x27;white&#x27;}</b>
<b>+┊   ┊134┊            borderRadius&#x3D;{12}</b>
<b>+┊   ┊135┊            color&#x3D;{&#x27;white&#x27;}</b>
<b>+┊   ┊136┊            iconStyle&#x3D;{styles.checkButtonIcon}</b>
<b>+┊   ┊137┊            name&#x3D;{&#x27;check&#x27;}</b>
<b>+┊   ┊138┊            onPress&#x3D;{this.toggle}</b>
<b>+┊   ┊139┊            size&#x3D;{16}</b>
<b>+┊   ┊140┊            style&#x3D;{styles.checkButton}</b>
<b>+┊   ┊141┊          /&gt;</b>
<b>+┊   ┊142┊        &lt;/View&gt;</b>
<b>+┊   ┊143┊      &lt;/View&gt;</b>
<b>+┊   ┊144┊    );</b>
<b>+┊   ┊145┊  }</b>
<b>+┊   ┊146┊}</b>
<b>+┊   ┊147┊Cell.propTypes &#x3D; {</b>
<b>+┊   ┊148┊  isSelected: PropTypes.func,</b>
<b>+┊   ┊149┊  item: PropTypes.shape({</b>
<b>+┊   ┊150┊    username: PropTypes.string.isRequired,</b>
<b>+┊   ┊151┊  }).isRequired,</b>
<b>+┊   ┊152┊  toggle: PropTypes.func.isRequired,</b>
<b>+┊   ┊153┊};</b>
<b>+┊   ┊154┊</b>
<b>+┊   ┊155┊class NewGroup extends Component {</b>
<b>+┊   ┊156┊  static navigationOptions &#x3D; ({ navigation }) &#x3D;&gt; {</b>
<b>+┊   ┊157┊    const { state } &#x3D; navigation;</b>
<b>+┊   ┊158┊    const isReady &#x3D; state.params &amp;&amp; state.params.mode &#x3D;&#x3D;&#x3D; &#x27;ready&#x27;;</b>
<b>+┊   ┊159┊    return {</b>
<b>+┊   ┊160┊      title: &#x27;New Group&#x27;,</b>
<b>+┊   ┊161┊      headerRight: (</b>
<b>+┊   ┊162┊        isReady ? &lt;Button</b>
<b>+┊   ┊163┊          title&#x3D;&quot;Next&quot;</b>
<b>+┊   ┊164┊          onPress&#x3D;{state.params.finalizeGroup}</b>
<b>+┊   ┊165┊        /&gt; : undefined</b>
<b>+┊   ┊166┊      ),</b>
<b>+┊   ┊167┊    };</b>
<b>+┊   ┊168┊  };</b>
<b>+┊   ┊169┊</b>
<b>+┊   ┊170┊  constructor(props) {</b>
<b>+┊   ┊171┊    super(props);</b>
<b>+┊   ┊172┊</b>
<b>+┊   ┊173┊    let selected &#x3D; [];</b>
<b>+┊   ┊174┊    if (this.props.navigation.state.params) {</b>
<b>+┊   ┊175┊      selected &#x3D; this.props.navigation.state.params.selected;</b>
<b>+┊   ┊176┊    }</b>
<b>+┊   ┊177┊</b>
<b>+┊   ┊178┊    this.state &#x3D; {</b>
<b>+┊   ┊179┊      selected: selected || [],</b>
<b>+┊   ┊180┊      friends: props.user ?</b>
<b>+┊   ┊181┊        _.groupBy(props.user.friends, friend &#x3D;&gt; friend.username.charAt(0).toUpperCase()) : [],</b>
<b>+┊   ┊182┊    };</b>
<b>+┊   ┊183┊</b>
<b>+┊   ┊184┊    this.finalizeGroup &#x3D; this.finalizeGroup.bind(this);</b>
<b>+┊   ┊185┊    this.isSelected &#x3D; this.isSelected.bind(this);</b>
<b>+┊   ┊186┊    this.toggle &#x3D; this.toggle.bind(this);</b>
<b>+┊   ┊187┊  }</b>
<b>+┊   ┊188┊</b>
<b>+┊   ┊189┊  componentDidMount() {</b>
<b>+┊   ┊190┊    this.refreshNavigation(this.state.selected);</b>
<b>+┊   ┊191┊  }</b>
<b>+┊   ┊192┊</b>
<b>+┊   ┊193┊  componentWillReceiveProps(nextProps) {</b>
<b>+┊   ┊194┊    const state &#x3D; {};</b>
<b>+┊   ┊195┊    if (nextProps.user &amp;&amp; nextProps.user.friends &amp;&amp; nextProps.user !&#x3D;&#x3D; this.props.user) {</b>
<b>+┊   ┊196┊      state.friends &#x3D; sortObject(</b>
<b>+┊   ┊197┊        _.groupBy(nextProps.user.friends, friend &#x3D;&gt; friend.username.charAt(0).toUpperCase()),</b>
<b>+┊   ┊198┊      );</b>
<b>+┊   ┊199┊    }</b>
<b>+┊   ┊200┊</b>
<b>+┊   ┊201┊    if (nextProps.selected) {</b>
<b>+┊   ┊202┊      Object.assign(state, {</b>
<b>+┊   ┊203┊        selected: nextProps.selected,</b>
<b>+┊   ┊204┊      });</b>
<b>+┊   ┊205┊    }</b>
<b>+┊   ┊206┊</b>
<b>+┊   ┊207┊    this.setState(state);</b>
<b>+┊   ┊208┊  }</b>
<b>+┊   ┊209┊</b>
<b>+┊   ┊210┊  componentWillUpdate(nextProps, nextState) {</b>
<b>+┊   ┊211┊    if (!!this.state.selected.length !&#x3D;&#x3D; !!nextState.selected.length) {</b>
<b>+┊   ┊212┊      this.refreshNavigation(nextState.selected);</b>
<b>+┊   ┊213┊    }</b>
<b>+┊   ┊214┊  }</b>
<b>+┊   ┊215┊</b>
<b>+┊   ┊216┊  refreshNavigation(selected) {</b>
<b>+┊   ┊217┊    const { navigation } &#x3D; this.props;</b>
<b>+┊   ┊218┊    navigation.setParams({</b>
<b>+┊   ┊219┊      mode: selected &amp;&amp; selected.length ? &#x27;ready&#x27; : undefined,</b>
<b>+┊   ┊220┊      finalizeGroup: this.finalizeGroup,</b>
<b>+┊   ┊221┊    });</b>
<b>+┊   ┊222┊  }</b>
<b>+┊   ┊223┊</b>
<b>+┊   ┊224┊  finalizeGroup() {</b>
<b>+┊   ┊225┊    const { navigate } &#x3D; this.props.navigation;</b>
<b>+┊   ┊226┊    navigate(&#x27;FinalizeGroup&#x27;, {</b>
<b>+┊   ┊227┊      selected: this.state.selected,</b>
<b>+┊   ┊228┊      friendCount: this.props.user.friends.length,</b>
<b>+┊   ┊229┊      userId: this.props.user.id,</b>
<b>+┊   ┊230┊    });</b>
<b>+┊   ┊231┊  }</b>
<b>+┊   ┊232┊</b>
<b>+┊   ┊233┊  isSelected(user) {</b>
<b>+┊   ┊234┊    return ~this.state.selected.indexOf(user);</b>
<b>+┊   ┊235┊  }</b>
<b>+┊   ┊236┊</b>
<b>+┊   ┊237┊  toggle(user) {</b>
<b>+┊   ┊238┊    const index &#x3D; this.state.selected.indexOf(user);</b>
<b>+┊   ┊239┊    if (~index) {</b>
<b>+┊   ┊240┊      const selected &#x3D; update(this.state.selected, { $splice: [[index, 1]] });</b>
<b>+┊   ┊241┊</b>
<b>+┊   ┊242┊      return this.setState({</b>
<b>+┊   ┊243┊        selected,</b>
<b>+┊   ┊244┊      });</b>
<b>+┊   ┊245┊    }</b>
<b>+┊   ┊246┊</b>
<b>+┊   ┊247┊    const selected &#x3D; [...this.state.selected, user];</b>
<b>+┊   ┊248┊</b>
<b>+┊   ┊249┊    return this.setState({</b>
<b>+┊   ┊250┊      selected,</b>
<b>+┊   ┊251┊    });</b>
<b>+┊   ┊252┊  }</b>
<b>+┊   ┊253┊</b>
<b>+┊   ┊254┊  render() {</b>
<b>+┊   ┊255┊    const { user, loading } &#x3D; this.props;</b>
<b>+┊   ┊256┊</b>
<b>+┊   ┊257┊    // render loading placeholder while we fetch messages</b>
<b>+┊   ┊258┊    if (loading || !user) {</b>
<b>+┊   ┊259┊      return (</b>
<b>+┊   ┊260┊        &lt;View style&#x3D;{[styles.loading, styles.container]}&gt;</b>
<b>+┊   ┊261┊          &lt;ActivityIndicator /&gt;</b>
<b>+┊   ┊262┊        &lt;/View&gt;</b>
<b>+┊   ┊263┊      );</b>
<b>+┊   ┊264┊    }</b>
<b>+┊   ┊265┊</b>
<b>+┊   ┊266┊    return (</b>
<b>+┊   ┊267┊      &lt;View style&#x3D;{styles.container}&gt;</b>
<b>+┊   ┊268┊        {this.state.selected.length ? &lt;View style&#x3D;{styles.selected}&gt;</b>
<b>+┊   ┊269┊          &lt;SelectedUserList</b>
<b>+┊   ┊270┊            data&#x3D;{this.state.selected}</b>
<b>+┊   ┊271┊            remove&#x3D;{this.toggle}</b>
<b>+┊   ┊272┊          /&gt;</b>
<b>+┊   ┊273┊        &lt;/View&gt; : undefined}</b>
<b>+┊   ┊274┊        {_.keys(this.state.friends).length ? &lt;AlphabetListView</b>
<b>+┊   ┊275┊          style&#x3D;{{ flex: 1 }}</b>
<b>+┊   ┊276┊          data&#x3D;{this.state.friends}</b>
<b>+┊   ┊277┊          cell&#x3D;{Cell}</b>
<b>+┊   ┊278┊          cellHeight&#x3D;{30}</b>
<b>+┊   ┊279┊          cellProps&#x3D;{{</b>
<b>+┊   ┊280┊            isSelected: this.isSelected,</b>
<b>+┊   ┊281┊            toggle: this.toggle,</b>
<b>+┊   ┊282┊          }}</b>
<b>+┊   ┊283┊          sectionListItem&#x3D;{SectionItem}</b>
<b>+┊   ┊284┊          sectionHeader&#x3D;{SectionHeader}</b>
<b>+┊   ┊285┊          sectionHeaderHeight&#x3D;{22.5}</b>
<b>+┊   ┊286┊        /&gt; : undefined}</b>
<b>+┊   ┊287┊      &lt;/View&gt;</b>
<b>+┊   ┊288┊    );</b>
<b>+┊   ┊289┊  }</b>
<b>+┊   ┊290┊}</b>
<b>+┊   ┊291┊</b>
<b>+┊   ┊292┊NewGroup.propTypes &#x3D; {</b>
<b>+┊   ┊293┊  loading: PropTypes.bool.isRequired,</b>
<b>+┊   ┊294┊  navigation: PropTypes.shape({</b>
<b>+┊   ┊295┊    navigate: PropTypes.func,</b>
<b>+┊   ┊296┊    setParams: PropTypes.func,</b>
<b>+┊   ┊297┊    state: PropTypes.shape({</b>
<b>+┊   ┊298┊      params: PropTypes.object,</b>
<b>+┊   ┊299┊    }),</b>
<b>+┊   ┊300┊  }),</b>
<b>+┊   ┊301┊  user: PropTypes.shape({</b>
<b>+┊   ┊302┊    id: PropTypes.number,</b>
<b>+┊   ┊303┊    friends: PropTypes.arrayOf(PropTypes.shape({</b>
<b>+┊   ┊304┊      id: PropTypes.number,</b>
<b>+┊   ┊305┊      username: PropTypes.string,</b>
<b>+┊   ┊306┊    })),</b>
<b>+┊   ┊307┊  }),</b>
<b>+┊   ┊308┊  selected: PropTypes.arrayOf(PropTypes.object),</b>
<b>+┊   ┊309┊};</b>
<b>+┊   ┊310┊</b>
<b>+┊   ┊311┊const userQuery &#x3D; graphql(USER_QUERY, {</b>
<b>+┊   ┊312┊  options: (ownProps) &#x3D;&gt; ({ variables: { id: 1 } }), // fake for now</b>
<b>+┊   ┊313┊  props: ({ data: { loading, user } }) &#x3D;&gt; ({</b>
<b>+┊   ┊314┊    loading, user,</b>
<b>+┊   ┊315┊  }),</b>
<b>+┊   ┊316┊});</b>
<b>+┊   ┊317┊</b>
<b>+┊   ┊318┊export default compose(</b>
<b>+┊   ┊319┊  userQuery,</b>
<b>+┊   ┊320┊)(NewGroup);</b>
</pre>

##### Changed server&#x2F;data&#x2F;resolvers.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊26┊26┊        groupId,
 ┊27┊27┊      });
 ┊28┊28┊    },
<b>+┊  ┊29┊    createGroup(_, { name, userIds, userId }) {</b>
<b>+┊  ┊30┊      return User.findOne({ where: { id: userId } })</b>
<b>+┊  ┊31┊        .then(user &#x3D;&gt; user.getFriends({ where: { id: { $in: userIds } } })</b>
<b>+┊  ┊32┊          .then(friends &#x3D;&gt; Group.create({</b>
<b>+┊  ┊33┊            name,</b>
<b>+┊  ┊34┊            users: [user, ...friends],</b>
<b>+┊  ┊35┊          })</b>
<b>+┊  ┊36┊            .then(group &#x3D;&gt; group.addUsers([user, ...friends])</b>
<b>+┊  ┊37┊              .then(() &#x3D;&gt; group),</b>
<b>+┊  ┊38┊            ),</b>
<b>+┊  ┊39┊          ),</b>
<b>+┊  ┊40┊        );</b>
<b>+┊  ┊41┊    },</b>
<b>+┊  ┊42┊    deleteGroup(_, { id }) {</b>
<b>+┊  ┊43┊      return Group.find({ where: id })</b>
<b>+┊  ┊44┊        .then(group &#x3D;&gt; group.getUsers()</b>
<b>+┊  ┊45┊          .then(users &#x3D;&gt; group.removeUsers(users))</b>
<b>+┊  ┊46┊          .then(() &#x3D;&gt; Message.destroy({ where: { groupId: group.id } }))</b>
<b>+┊  ┊47┊          .then(() &#x3D;&gt; group.destroy()),</b>
<b>+┊  ┊48┊        );</b>
<b>+┊  ┊49┊    },</b>
<b>+┊  ┊50┊    leaveGroup(_, { id, userId }) {</b>
<b>+┊  ┊51┊      return Group.findOne({ where: { id } })</b>
<b>+┊  ┊52┊        .then(group &#x3D;&gt; group.removeUser(userId)</b>
<b>+┊  ┊53┊          .then(() &#x3D;&gt; group.getUsers())</b>
<b>+┊  ┊54┊          .then((users) &#x3D;&gt; {</b>
<b>+┊  ┊55┊            // if the last user is leaving, remove the group</b>
<b>+┊  ┊56┊            if (!users.length) {</b>
<b>+┊  ┊57┊              group.destroy();</b>
<b>+┊  ┊58┊            }</b>
<b>+┊  ┊59┊            return { id };</b>
<b>+┊  ┊60┊          }),</b>
<b>+┊  ┊61┊        );</b>
<b>+┊  ┊62┊    },</b>
<b>+┊  ┊63┊    updateGroup(_, { id, name }) {</b>
<b>+┊  ┊64┊      return Group.findOne({ where: { id } })</b>
<b>+┊  ┊65┊        .then(group &#x3D;&gt; group.update({ name }));</b>
<b>+┊  ┊66┊    },</b>
 ┊29┊67┊  },
 ┊30┊68┊  Group: {
 ┊31┊69┊    users(group) {
</pre>

##### Changed server&#x2F;data&#x2F;schema.js
<pre>
<i>╔══════╗</i>
<i>║ diff ║</i>
<i>╚══════╝</i>
 ┊49┊49┊    createMessage(
 ┊50┊50┊      text: String!, userId: Int!, groupId: Int!
 ┊51┊51┊    ): Message
<b>+┊  ┊52┊    createGroup(name: String!, userIds: [Int], userId: Int!): Group</b>
<b>+┊  ┊53┊    deleteGroup(id: Int!): Group</b>
<b>+┊  ┊54┊    leaveGroup(id: Int!, userId: Int!): Group # let user leave group</b>
<b>+┊  ┊55┊    updateGroup(id: Int!, name: String): Group</b>
 ┊52┊56┊  }
 ┊53┊57┊  
 ┊54┊58┊  schema {
</pre>

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

⟸ <a href="https://github.com/srtucker22/chatty/tree/master@2.0.0/.tortilla/manuals/views/medium/step3.md">PREVIOUS STEP</a> <b>║</b> <a href="https://github.com/srtucker22/chatty/tree/master@2.0.0/.tortilla/manuals/views/medium/step5.md">NEXT STEP</a> ⟹

[}]: #
