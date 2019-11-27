# Text commands in `status-react`

In the past Status had chat commands that the user would invoke using the `/` prefix followed by the command's name. These commands were enabling complex features that go beyond the usual text messages, such as sending or requesting funds to contacts. At the time these two commands were easy to implement because the user's wallet and chat accounts were the same, which means the wallet address could be derived from the public key of this account's keypair.

Then we moved to the multi-account structure, to integrate the keycard in Status and provide more privacy to the users. The chat private key needs to be loaded in memory to decrypt messages as they come in, but for security reasons, we wanted to avoid that with the wallet private key. We were also concerned about the privacy issues that came with having a wallet address that can be derived from the public key used in chats because everyone was able to check out the users' balances. So now the chat and wallet accounts are two different accounts with different keypairs, and one cannot derive the wallet address from the public key used in chats anymore. Additionally, users can now create as many wallets as they want within a multi-account.

For this reason, the send and request commands cannot work as they used to anymore because the user needs to choose which wallet he wants to send from, and he needs the recipient to communicate which wallet address he wants to receive the funds on. So we disabled the send and request commands temporarily to figure out a good user experience.

As the community feedbacks suggest, these two commands were particularly appreciated and were considered one of the killer features of Status so we should re-enable them before v1 release.

In this document, we are going to go through the send command implementation and propose an alternative.

# Send, a `PersonalSendCommand`

The `PersonalSendCommand` is the type of the send command in one to one chats, defined in `status-im.chat.commands.impl.transactions`. It implements the `Command` protocol, the `Yielding` protocol and the `EhancedParameters` protocol, which are all defined in `status-im.chat.commands.protocol`. 

> In Clojure protocols are just a set of methods and signatures, and implementing a protocol is just like saying: "types that implement this protocol all have a bunch of methods with those names and this number of parameters". It prevents you from implementing more methods than the protocol defines but it does not protect you against types that don't implement all the methods of the protocol. Another way of putting it is that you would get a compile-time error if you mistype a method or implement it with the wrong arity, but only a runtime error if you tried to call a method of the protocol on the instance of a type implementing that protocol but not that particular method. If I have a protocol `Foo` with a method `bar`, I can define a type `Foobar` that implements `Foo` but not `bar` and only get an exception at runtime if I try to call `bar` on an instance of `Foobar`, but if I spell the method `baz` when defining `Foobar`, I get a compile-time error.

Types and protocols aren't very common in Clojure code, they are usually used in libraries or at the edge of systems. In this case, we also define a type only to instantiate it once in `status-im.chat.commands.core/register` and never again.

If we look only at the "Command" protocol, it is defined in `status-im.chat.commands.protocol/Command`. It has an `id`, `scope`, `description`, `parameter` methods than are simply expected to return data and could, therefore, be replaced by a map describing these data. These could be validated at compile time or runtime for dynamically loaded commands (in the case of extensions), unlike the current method implementations. It also has 5 more methods, `validate`, `on-send`, `on-receive`, `short-preview` `preview`, the first 3 are used when sending/receiving, the last 2 when rendering.

## The effects of the '/'

When you type `/` or tap on `/`, the list of available commands pops up above the input field. The only 2 available commands are `send` and `receive` in one to one chat and there is none in a group chat and public chat, so typing `/` won't pop anything and you can't tap the `/` icon because it is not there.

When running in a dev environment, the re-frame event flow is the easiest mean to understand the app, with the monitoring tool re-frisk available on http://localhost:4567/#, or with a simple REPL at port 8777 to inspect the app state in `re-frame.db/app-db` , dispatch events with `re-frame.core/dispatch` and subscribe to data derived from the app state with `re-frame.core/subscribe`. Therefore we are going to analyze the current command flow from 2 perspectives: events and subscriptions.

## Events

Typing `/` in the input field dispatches a `chat.ui/set-chat-input-text` event while tapping the `/` icon dispatches a `chat.ui/set-command-prefix` event.
In the app-state, both events have the same effect of setting the `input-text` of the current chat to "/". That is because both are using the `set-chat-input-text` function which produces this effect. Additionally, the event dispatched when tapping the icon calls the `chat-input-focus` function which has the side effect of focusing the app on the input field, so that the keyboard pops out and the user can start typing the command name.

3 `chat.ui/set-chat-ui-props` events are then dispatched for no obvious reason, two setting the `input-ref` with no effect (was already set) and one setting the `selection` to 1, which is the position of the caret in the input field. Since the only effect of setting the selection could be produced by the `set-chat-input-text` function. The fact that these events pop so often when interacting with the input field is an issue by itself, events are supposed to mean something and these setter events are meaningless.

Surprisingly that is all that happens event wise. This means that we are going to need to dig into subscriptions, the mere effect of setting the `input-text` to `/` in the state had the cascading effect of showing up a list with a couple of commands in the UI, and that is typically what subscriptions are for.

## Subscriptions

We have 6 subscriptions that contain "command" in their keyword.

### `status-im.subs/access-scope->command-id` and `chats/id->command`

`:status-im.subs/access-scope->command-id` and `chats/id->command` are root subscriptions, which means that the keys at the top level of the `app-db` which is a map. They are added to `app-db` early in the app startup process, as an effect of the `status-im.chat.commands.core/index-commands` function. `chats/id->command` given a `command-id` which is a tuple of command id and scope, returns a `command` map with a `type` and `params` key. `type` here is misleading because it is actually an instance of the type of the command, as defined in a `deftype`. 

### `status-im.subs/get-commands-for-chat`

```clojure
(re-frame/reg-sub
 ::get-commands-for-chat
 :<- [:chats/id->command]
 :<- [::access-scope->command-id]
 :<- [:chats/current-chat]
 (fn [[id->command access-scope->command-id chat]]
   (commands/chat-commands id->command access-scope->command-id chat)))
```

> `re-frame/reg-sub` is the way to register a subscription in re-frame. Each subscription is named with a keyword that is used to subscribe to it and contains a function that is called when the inputs of the subscription are changed. Root subscriptions will receive the app-db as input, while others will receive the output of other subscriptions which forms an acyclic graph of computations (acyclicity is not enforced though, TO TEST does it blow up the app?), with app-db at the root. The advantage is that the outputs can be reused as input for multiple subscriptions, and only subscriptions whose output has changed are recomputed. `chats/current-chat` producing 


In the code above, `get-commands-for-chat` receives the output of the `chat/id->command`, `:status-im.subs/access-scope->command-id` and `chats/current-chat`. We already identified the first 2 as root subscriptions. `:chats/current-chat` is a much more complex subscription. It is already computed because the chat screen subscribes to it, but only the `chat-id`, `group-chat` and `public` fields of the current-chat subscription output are used, and since the recomputation of a subscription is triggered by any change to its inputs, even the price of a currency, even the smallest change, even to fields that are unused in `get-command-chat`, is going to trigger a recomputation. The output of the subscription is a map of id to commands for the current chat.

> In re-frame/reagent/react if a subscription is recomputed, which happens whenever one of its inputs has changed, then all the components that are dereferencing the subscriptions are going to re-render.

In this case, it may or may not be a big deal, but ideally, we should avoid these unnecessary rerenders. They can produce flickering in the UI and are just generally wasting computations.

### `chats/all-available-commands`, `chats/available-commands` and `chats/selected-chat-command`

`chats/all-available-commands` is only taking `get-commands-for-chat` as input and simply take the values of that map and order them by name.

`chats/available-commands` takes `get-commands-for-chat` and `chats/current-chat`. Only the `input-text` field of the `chats/current-chat` subscription output is actually used to filter command names based on that.

Similarly `chats/selected-chat-command` takes `chats/current-chat` but only uses `input-text` as input. It also takes `get-commands-for-chat` and `chats/current-chat-ui-prop :selection`. Its output is computed by the `selected-chat-command` which returns the command props as well as some extra derived data for the UX such as indexed parameters, `current-param-position` and `command-completion` (whose value depends on the number of expected input parameters compared to the others).

## Sending the command

A lot is going on already but at this point, all we've done is select a command and get ready to enter parameters. From there the autocompletion of assets, the first parameter, is broken because the structure of the wallet has changed and isn't a single account anymore.

Once we have entered all parameters and they are considered valid, we can press the send command. It is only a soft validation, as one can enter an amount bigger than the account holds. This is because there is a second validation step when signing the transaction, which will warn the user that he owns insufficient funds.

A very confusing part of commands namespaces now is the `status-im.chat.commands.sending` and it's `status-im.chat.commands.receiving` counterpart. As mentioned earlier there are only two actual commands, `send` and `receive`, and these namespaces have nothing to do with them. It is the action of sending and receiving a command that is defined here. For the `send` command, 

`status-im.chat.commands.sending/validate-and-send` is called when the user presses the `send` button in the chat input, while `send` is called once the transaction has been sent. Because the `send` command implements the `Yielding` protocol, it calls `yield-control` after the validation, which will initiate the sign-transaction flow, after which the `status-im.chat.commands.sending/send` function is finally called by the `:chat/send-transaction-result` event.

## Send message

In the `create-command-message` of the `status-im.chat.commands.sending` namespace we can find the following:

```clojure
 {:chat-id      chat-id
     :content-type constants/content-type-command
     :content      {:chat-id      chat-id
                    :command-path command-path
                    :params       params}}
```

From the definition of the `PersonalSendCommand` type in `status-im.chat.commands.impl.transactions` we know that the params of the send command are the same as the request command:

```clojure
(def personal-send-request-params
  [{:id          :asset
    :type        :text
    :placeholder (i18n/label :t/send-request-currency)
    :suggestions choose-asset-suggestion}
   {:id          :amount
    :type        :number
    :placeholder (i18n/label :t/send-request-amount)}])
```

However, it also implements the `EhancedParameters` protocol, and adds `chain`, `network`, `fiat-amount` and `currency`.

I am not sure why it adds any of those parameters except for `network`.

Aditdionally one has to dig into `status-im.chat.models.input/send-transaction-result` to realize that `tx-hash`, the transaction-hash, is added to the parameter-map as well after signing the transaction.

## Conclusion

We have only gone through the sending of the send command, which means we could also explore the receiving and the `receive` command. However, there is no point in spending more time on this code as it should be deprecated. The cost of fixing it is greater than the implementation of the GUI commands which will provide a better user experience. The entire `chat.command` namespaces will be deleted in favor of a much simpler implementation.

The send and receive flow will share screens with the wallet transaction flow, as well as most subscriptions and events, which means that the code strictly related to commands themselves SHOULD be minimal. The schemas of the send and receive messages, which currently have to be inferred from the code MUST be explicitly defined and specified in a follow-up document.
