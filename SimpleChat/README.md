![BlackBerry Spark Communications Services](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/resources/images/bnr-bbm-enterprise-sdk-title.png)
# SimpleChat for Android

The Simple Chat example demonstrates how you can build a simple chat
application using BlackBerry Spark Communications Services.  This example
demonstrates how easily messaging can be integrated into your application.  For
a more rich chat app experience please see the
[Rich Chat](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/examples/android/RichChat/README.html)
app provided with the SDK. This example builds on the
[Quick Start](../QuickStart/README.md) example that demonstrates how you can
authenticate with the SDK using the
[Identity Provider](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/identityManagement.html)
of your application.

### Features

It allows the user to do the following:

* Create a chat
* View the chat list
* View all sent and received messages in a chat
* Send text-based messages
* Mark incoming messages as Read

<br>

<p align="center">
<a href="screenShots/chat_list.png"><img src="screenShots/chat_list.png" width="25%" height="25%"></a> 
<a href="screenShots/chat.png"><img src="screenShots/chat.png" width="25%" height="25%"></a> 
</p>

## Getting Started

This example requires the Spark Communications SDK, which you can find along with related resources at the locations below.

* Instructions to
[Download and Configure](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/gettingStarted.html)
the SDK.
* [Android Getting Started](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/gettingStarted-android.html)
instructions in the Developer Guide.
* [API Reference](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/index.html)
<p align="center">
    <a href="http://www.youtube.com/watch?feature=player_embedded&v=310UDOFCLWM"
      target="_blank"><img src="../QuickStart/screenShots/bbme-sdk-android-getting-started.jpg"
      alt="YouTube Getting Started Video" width="486" height="" border="364"/></a>
</p>
<p align="center">
 <b>Getting started video</b>
</p>

By default, this example application is configured to work in a domain with user
authentication disabled and the BlackBerry Key Management Service enabled.
See the [Download & Configure](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/gettingStarted.html)
section of the Developer Guide to get started configuring a
[domain](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/faq.html#domain)
in the [sandbox](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/faq.html#sandbox).

Once you have a domain in the sandbox, edit Simple Chat's `app.properties` file
to configure the example with your domain ID.

```
# Your Spark Domain ID
user_domain="your_spark_domain"
```
When you run the Simple Chat application, it will prompt you for a user ID and
a password. Since you've configured your domain to have user authentication
disabled, you can enter any string you like for the user ID and an SDK identity
will be created for it. Other applications that you run in the same domain will
be able to find this identity by this user ID. The password is used to protected
the keys stored in the
[BlackBerry Key Management Service](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/security.html).

Notes:

* To complete a release build you must create your own signing key. To create your own signing key, visit https://developer.android.com/studio/publish/app-signing.html.
* This application has been built using gradle 4.2.1 (newer versions have not been validated).

## Walkthrough

Follow this guide for a walkthrough showing how the SDK is used to demonstrate simple messaging in this sample application.

* [SDK Initialization](#sdkSetup)
* [Getting chats](#gettingChats)
* [Starting a chat](#startingAChat)
* [Getting chat messages](#gettingChatMessages)
* [Displaying chat messages](#displayChatMessages)
* [Sending a chat message](#sendChatMessage)
* [Marking messages as read](#markAsRead)


### <a name="sdkSetup"></a>SDK Initialization

To use the BlackBerry Spark Communications Services SDK in your application you need to initialize and start the sdk.

```java
// Initialize BBMEnterprise SDK then start it
BBMEnterprise.getInstance().initialize(this);
BBMEnterprise.getInstance().start();
```
*MainActivity.java*

The SDK will encrypt your messages for you. The SimpleChat example uses the
BlackBerry Key Management System to store the security keys generated by the
SDK. The specifics of the key distribution are not described in this
guide. For more information visit the
[Protect](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/security.html)
guide and check out the example implementation in the
[Support library](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/examples/android/Support/README.html).


```java
KeySource keySource = new BlackBerryKMSSource(new UserChallengePasscodeProvider(context));
KeySourceManager.setKeySource(keySource);
keySource.start();
```

This sample uses a mock identity provider to create unsigned JWT tokens. For more information see [Identity Management](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/identityManagement.html) and the [Quick Start sample](../QuickStart/README.md). The MockTokenProvider class from the support library is used to generate the tokens.

We monitor the
[GlobalSetupState](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/GlobalSetupState.html)
to handle key states. First we add an
[Observer](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/reactive/Observer.html)
to the
[GlobalSetupState](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/GlobalSetupState.html)
[ObservableValue](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/reactive/ObservableValue.html). When
the GlobalSetupState changes our observers changed() method will be called. If
the setup state is:


* `DeviceSwitchRequired` we will ask the SDK to switch to this device by sending a [SetupRetry](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/outbound/SetupRetry.html) message.
* `NotRequested` we will register the local device by sending a [EndpointUpdate](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/outbound/EndpointUpdate.html).
* `Full` we will need to deregister a device using [EndpointDeregister](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/outbound/EndpointDeregister.html). If successful then send a [SetupRetry](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/outbound/SetupRetry.html) message to continue setup.

```java
//Listen to the setup events
final ObservableValue<GlobalSetupState> globalSetupState = BBMEnterprise.getInstance().getBbmdsProtocol().getGlobalSetupState();
mBbmSetupObserver = new Observer() {
    @Override
    public void changed() {
        final ObservableValue<GlobalSetupState> globalSetupState = BBMEnterprise.getInstance().getBbmdsProtocol().getGlobalSetupState();

        if (globalSetupState.get().exists != Existence.YES) {
            return;
        }

        GlobalSetupState currentState = globalSetupState.get();

        switch (currentState.state) {
            case NotRequested:
                SetupHelper.registerDevice("Simple Chat", "Simple Chat example");
                break;
            case DeviceSwitchRequired:
                //Ask the BBM Enterprise SDK to move the users profile to this device
                BBMEnterprise.getInstance().getBbmdsProtocol().send(new SetupRetry());
                break;
            case Full:
                SetupHelper.handleFullState();
                break;
            case Ongoing:
            case Success:
            case Unspecified:
                break;
        }
    }
};

//Add setup observer to the globalSetupStateObservable
globalSetupState.addObserver(mBbmSetupObserver);

//Call changed to trigger our observer to run immediately
mBbmSetupObserver.changed();
```



### <a name="gettingChats"></a>Getting chats

The
[`chatList`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/BbmdsProtocol.html#getChatList--)
is provided from the SDK as an
[`ObservableList`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/internal/lists/ObservableList). To
track changes to the chat list we register an
[`IncrementalListObserver`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/internal/lists/IncrementalListObserver)
with the chats list. Our `IncrementalListObserver` will be informed when the chat
list is modified. We pass those change notifications on to a
RecyclerView.Adapter which displays the chat list.


```java
//This observer will be used to notify the adapter when chats have been changed or added.
private final IncrementalListObserver mChatListObserver = new IncrementalListObserver() {
    @Override
    public void onItemsInserted(int position, int itemCount) {
        mAdapter.notifyItemRangeInserted(position, itemCount);
    }

    @Override
    public void onItemsRemoved(int position, int itemCount) {
        mAdapter.notifyItemRangeRemoved(position, itemCount);
    }

    @Override
    public void onItemsChanged(int position, int itemCount) {
        mAdapter.notifyItemRangeChanged(position, itemCount);
    }

    @Override
    public void onDataSetChanged() {
        mAdapter.notifyDataSetChanged();
    }
};

//Get the chat list and keep a hard reference to it
mChatList = BBMEnterprise.getInstance().getBbmdsProtocol().getChatList();
//Add our incremental list observer to the chat list
mChatList.addIncrementalListObserver(mChatListObserver);

//Set the adapter in the recyclerview
final RecyclerView chatsRecyclerView = (RecyclerView)findViewById(R.id.chats_list);
chatsRecyclerView.setAdapter(mAdapter);
chatsRecyclerView.setLayoutManager(new LinearLayoutManager(MainActivity.this));
```

*MainActivity.java*

To display the chats we are using a simple `RecyclerView.Adapter` and `ViewHolder`. For this example each item in the list displays the subject of the [`chat`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/Chat.html). When a user clicks on one of the chats we launch the `ChatActivity`.

```java
//Our chats recycler view adapter
private final RecyclerView.Adapter<ChatViewHolder> mAdapter = new RecyclerView.Adapter<ChatViewHolder>() {
    @Override
    public ChatViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View chatView = LayoutInflater.from(MainActivity.this).inflate(R.layout.chat_item, parent, false);
        return new ChatViewHolder(chatView);
    }

    @Override
    public void onBindViewHolder(ChatViewHolder holder, int position) {
        if (!mChatList.isPending()) {
            holder.title.setText(mChatList.get(position).subject);
            holder.chat = mChatList.get(position);
        }
    }

    @Override
    public int getItemCount() {
        return mChatList.size();
    }
};

//Simple view holder to display a chat
private class ChatViewHolder extends RecyclerView.ViewHolder {

    TextView title;
    Chat chat;

    ChatViewHolder(View itemView) {
        super(itemView);
        title = (TextView)itemView.findViewById(R.id.chat_title);

        //when the chat is clicked open the chat in a new activity
        itemView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent(MainActivity.this, ChatActivity.class);
                intent.putExtra("chat-id",  chat.chatId);
                startActivity(intent);
            }
        });
    }
}
```

### <a name="startingAChat"></a>Starting a chat

To start a chat you first need to indentify who you want to chat with. The SDK
provides a unique `regId` for every user who registers. The SDK keeps a
mapping of your application user identifier and `regIds`. See the
[identity management](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/identityManagement.html)
guide for more information.

This example adds a menu item which creates a dialog with input fields for a
user id and chat subject. The `regId` for a user can be found using the
[`IdentitiesGet`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/outbound/IdentitiesGet.html)
message. We are using the UserIdentityMapper class from the support library to
get an [`Observable`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/reactive/Observable.html)
map result. If the map result is found we can use the `regId` to start the chat.

```java
SingleshotMonitor.run(new SingleshotMonitor.RunUntilTrue() {
    @Override
    public boolean run() {
        //Lookup the Spark registration id for the provided user id
        UserIdentityMapper.IdentityMapResult mapResult =
                UserIdentityMapper.getInstance().getRegIdForUid(userId, true).get();
        if (mapResult.existence == Existence.MAYBE) {
             return false;
        }
        if (mapResult.existence == Existence.YES) {
            String subject = TextUtils.isEmpty(secondaryInput) ? userId : secondaryInput;
            startChat(mapResult.regId, subject);
        } else {
            Toast.makeText(MainActivity.this, getString(R.string.user_id_not_found, userId), Toast.LENGTH_LONG).show();
        }

        return false;
    }
});
```

This example only allows creating a chat with a single participant, but the SDK supports chats with up to 250 participants. To start a chat we send a [`ChatStart`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/outbound/ChatStart.html) message to the SDK.

```java
//Create a cookie to track the chat creation
final String cookie = UUID.randomUUID().toString();

//Create the invitee using the registration id
ChatStart.Invitees invitee = new ChatStart.Invitees();
invitee.regId(regId);

//Ask the BBM Enterprise SDK to start a new chat with the invitee and subject provided.
BBMEnterprise.getInstance().getBbmdsProtocol().send(new ChatStart(cookie, Lists.newArrayList(invitee), subject));
```

To track the creation of the chat you need to register a
[`ProtocolMessageConsumer`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/service/ProtocolMessageConsumer.html). The
ProtocolMessageConsumer is notified of every message that the SDK sends to
the application. Use the cookie we sent with the `ChatStart` to process only
the response to your request.

If the message was of type
[`ChatStartFailed`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/inbound/ChatStartFailed.html)
the SDK was not able to start your chat. If the type was `listAdd` then the
SDK is returning us the chat that was created. You can parse the chat by using
the [`setAttributes()`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/internal/JsonConstructable.html)
method. Finally, launch the chat activity and attach the `chatId`.


```java
//Add a ProtocolMessageConsumer to track the creation of the chat.
BBMEnterprise.getInstance().getBbmdsProtocolConnector().addMessageConsumer(new ProtocolMessageConsumer() {
    @Override
    public void onMessage(ProtocolMessage message) {
        final JSONObject json = message.getData();
        Logger.d("onMessage: " + message);
        //If the cookie in the incoming message matches the cookie we provided
        //we know this message is the response to our chatStart request.
        if (cookie.equals(json.optString("cookie",""))) {
            //this is for us, stop listening
            BBMEnterprise.getInstance().getBbmdsProtocolConnector().removeMessageConsumer(this);


            if ("chatStartFailed".equals(message.getType())) {
                //If the message type is chatStartFailed the BBM Enterprise SDK was unable to create the chat
                ChatStartFailed chatStartFailed = new ChatStartFailed().setAttributes(message.getJSON());
                Logger.i("Failed to create chat with " + regId);
                Toast.makeText(MainActivity.this, "Failed to create chat for reason " + chatStartFailed.reason.toString(), Toast.LENGTH_LONG).show();
            } else if ("listAdd".equals(message.getType())) {
                //The chat was created successfully
                try {
                    final JSONArray elementsArray = json.getJSONArray("elements");
                    Chat chat = new Chat().setAttributes((JSONObject) elementsArray.get(0));
                    //Start our chat activity
                    Intent intent = new Intent(MainActivity.this, ChatActivity.class);
                    intent.putExtra("chat-id", chat.chatId);
                    startActivity(intent);
                } catch (final JSONException e) {
                    Logger.e(e, "Failed to process start chat message " + message);
                }
            }
        }
    }

    @Override
    public void resync() {
    }
});
```


### <a name="gettingChatMessages"></a>Getting chat messages

Now that you have started a chat you need to populate the chat messages. Chat
messages can be retrieved from the SDK by providing a
[`ChatMessageKey`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/ChatMessage.ChatMessageKey.html)
to
[`getChatMessage`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/BbmdsProtocol.html#getChatMessage-com.bbm.sdk.bbmds.ChatMessage.ChatMessageKey-). The
`ChatMessageKey` is a combination of the chat id and message id and uniquely
identifies a chat message. The valid set of message ids for a chat is given by
[[Chat.lastMessage](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/Chat.html#lastMessage) -
[`Chat.numMessages`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/Chat.html#numMessages)].

In this example to load messages we are using ChatMessageList from the [Support library](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/html/examples/android/Support/README.html) library. `ChatMessageList` is a utility we've created which simplifies lazy loading of chat messages. We use `ChatMessageList` and the `IncrementalListObserver` to populate a `RecyclerView.Adapter` just like we did with the chats list. We do need to tell the `ChatMessageList` when to start and stop monitoring the chat for new messages. We only monitor messages while the activity is resumed.

```java
//This observer will notify the adapter when chat messages are added or changed.
private IncrementalListObserver mMessageListObserver = new IncrementalListObserver() {
    @Override
    public void onItemsInserted(int position, int count) {
        mAdapter.notifyItemRangeInserted(position, count);
    }

    @Override
    public void onItemsRemoved(int position, int count) {
        mAdapter.notifyItemRangeRemoved(position, count);
    }

    @Override
    public void onItemsChanged(int position, int count) {
        mAdapter.notifyItemRangeChanged(position, count);
    }

    @Override
    public void onDataSetChanged() {
        mAdapter.notifyDataSetChanged();
    }
};

//Create the ChatMessageList
mChatMessageList = new ChatMessageList(mChatId);

//Initialize the recycler view
final RecyclerView messageRecyclerView = (RecyclerView) findViewById(R.id.messages_list);
messageRecyclerView.setAdapter(mAdapter);
LinearLayoutManager layoutManager = new LinearLayoutManager(ChatActivity.this);
messageRecyclerView.setLayoutManager(layoutManager);

//Add our IncrementalListObserver to the ChatMessageList
mChatMessageList.addIncrementalListObserver(mMessageListObserver);

@Override
protected void onPause() {
    super.onPause();
    //Stop loading messages from the ChatMessageList
    mChatMessageList.stop();
    //Mark all the messages as read when closing the chat
    markMessagesAsRead();
}

@Override
protected void onResume() {
    super.onResume();
    //Start loading messages from the ChatMessageList
    mChatMessageList.start();
    //Mark all the messages as read when opening the chat
    markMessagesAsRead();
}
```
*ChatActivity.java*


### <a name="displayChatMessages"></a>Displaying chat messages

The [`ChatMessages`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/ChatMessage.html) are text only in this example. But ChatMessage also supports file attachments, thumbnails, recall and custom data and types you can define.

To display the chat messages we provide our `Adapter` and `ViewHolder`. To left
and right align outgoing and incoming messages we're providing two different
types in our `Adapter`. Each chat message includes a set of
[`flags`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/ChatMessage.Flags.html)
we can check to find out if the message was incoming.

```java
@Override
public int getItemViewType(int position) {
    //Use the ChatMessage.Flag to determine if the message is incoming or outgoing and use the correct type
    ChatMessage message = mChatMessageList.get(position);
    return message.hasFlag(ChatMessage.Flags.Incoming) ? TYPE_INCOMING : TYPE_OUTGOING;
}

@Override
public MessageViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    //The incoming message layout is right justified, outgoing left justified
    int layoutRes = viewType == TYPE_INCOMING ? R.layout.incoming_message_item : R.layout.outgoing_message_item;
    View chatView = LayoutInflater.from(ChatActivity.this).inflate(layoutRes, parent, false);
    return new MessageViewHolder(chatView);
}
```
*ChatActivity.java*


Messages have a [`state`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/ChatMessage.State.html) which is used to notify users if their message has been sent, delivered or read. Incoming messages that are unread are bolded and outgoing messages have their state prepended. Finally, we add the content of the text message, or if the message not a Text message then display the name of the message type.
```java
@Override
public void onBindViewHolder(MessageViewHolder holder, int position) {
    //Get the message to display
    ChatMessage message = mChatMessageList.get(position);
    if (message.getExists() == Existence.MAYBE) {
        return;
    }
    String prefix = "";
    if (message.hasFlag(ChatMessage.Flags.Incoming)) {
        if (message.state != ChatMessage.State.Read) {
            //instead of displaying sent status like BBM, just show bold until it is read
            holder.messageText.setTypeface(null, Typeface.BOLD);
        } else {
            holder.messageText.setTypeface(null, Typeface.NORMAL);
        }
    } else {
        //show the state of the outgoing message
        switch (message.state) {
            case Sending:
                prefix = "(...) ";
                break;
            case Sent:
                prefix = "(S) ";
                break;
            case Delivered:
                prefix = "(D) ";
                break;
            case Read:
                prefix = "(R) ";
                break;
            case Failed:
                prefix = "(F) ";
                break;
            default:
                prefix = "(?) ";
        }
    }
    if (message.tag.equals(ChatMessage.Tag.Text)) {
        holder.messageText.setText(prefix + message.content);
    } else {
        //For non-text messages just display the message type Tag
        holder.messageText.setText(message.tag);
    }
}
```



### <a name="sendChatMessage"></a>Sending a chat message

Sending a new message in the chat is easy. Create a [`ChatMessageSend`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/outbound/ChatMessageSend.html) and the text the user typed as the content. You can also set a file, thumbnail and custom JSON data to the chat message.

```java
sendButton.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        String text = inputText.getText().toString();
        if (!text.isEmpty()) {
            //Send a new outgoing text message, setting the content to the input text
            BBMEnterprise.getInstance().getBbmdsProtocol().send(new ChatMessageSend(mChatId, ChatMessageSend.Tag.Text).content(text));
            inputText.setText("");
        }
    }
});
```

*ChatActivity.java*



### <a name="markAsRead"></a>Marking messages as read

You need to notify the  SDK of when a user has read a message. The SDK will propagate the status change to the other participants in the chat. To mark a message as read, send a [`ChatMessageRead`](https://developer.blackberry.com/files/bbm-enterprise/documents/guide/reference/android/com/bbm/sdk/bbmds/outbound/ChatMessageRead.html) to the SDK. All messages which are older than the message id provided are automatically marked as read. To keep this example simple, we are just sending a ChatMessageRead change with the lastMessage id. This example only sends message status changes when the chat activity is paused or resumed. You might want to choose a different action, like a marking a message as read when it becomes visible in the chat.

```java
/**
 * Mark all messages in the chat as read
 */
private void markMessagesAsRead() {
    final ObservableValue<Chat> obsChat = BBMEnterprise.getInstance().getBbmdsProtocol().getChat(mChatId);
    mMarkMessagesReadObserver = new Observer() {
        @Override
        public void changed() {
            final Chat chat = obsChat.get();
            if (chat.exists == Existence.YES) {
                //remove ourselves as an observer so we don't get triggered again
                obsChat.removeObserver(this);
                if (chat.numMessages == 0 || chat.lastMessage == 0) {
                    return;
                }

                //Ask the BBM Enterprise SDK to mark the last message in the chat as read.
                //All messages older then this will be marked as read.
                BBMEnterprise.getInstance().getBbmdsProtocol().send(
                        new ChatMessageRead(mChatId, chat.lastMessage)
                );
            }
        }
    };
    //Add the chatObserver to the chat
    obsChat.addObserver(mMarkMessagesReadObserver);
    //Run the changed method
    mMarkMessagesReadObserver.changed();
}
```
*ChatActivity.java*

## License

These samples are released as Open Source and licensed under the [Apache 2.0 License](http://www.apache.org/licenses/LICENSE-2.0.html).

The Android robot is reproduced or modified from work created and shared by Google and used according to terms described in the [Creative Commons 3.0 Attribution License](https://creativecommons.org/licenses/by/3.0/).

This page includes icons from: https://material.io/icons/ used under the [Apache 2.0 License](http://www.apache.org/licenses/LICENSE-2.0.html).

## Reporting Issues and Feature Requests

If you find an issue in one of the Samples or have a Feature Request, simply file an [issue](https://github.com/blackberry/bbme-sdk-android-samples/issues).

