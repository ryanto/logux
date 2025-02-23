# Application State

The state describes all data of your application. On the client-side state describe UI state, data load from the server, user’s local changes. On the server-side state is what you have in the database.


## Client State

Logux Redux uses *event sourcing* pattern on client-side. Actions describe the current state. State object is just an cache.

The developer provides reducer. The reducer has an initial state (`[]` in this example) and describes how to change the current state on this action:

```js
export default function usersReducer (state = [], action) {
  if (action.type === 'user/add') {
    return state.concat([action.user])
  } else if (action.type === 'user/remove') {
    return state.filter(user => user.id !== action.userId)
  }
}
```

The application combines reducers of each key in store:

```js
import { combineReducers } from 'redux'
import usersReducer from './users'

export default combineReducers({
  users: usersReducer
})
```

This big reducers generate initial state during Logux Redux initialization:

```js
reducer(undefined, { type: '@@redux/INIT' }) //=> { users: [] }
```

Then each action will change the state:

```js
const action = {
  type: 'user/add',
  user: { id: '7tEL8t', name: 'First' }
}
reducer({ users: [] }, action) //=> { users: [{ id: '7tEL8t', name: 'First' }] }
```

Each reducer must be a pure function (always return the same result on the same argument):

1. It should create a new state object, not changing the old one.
2. It should be state-less. It should work only with old state and action. You can not use `localStorage` or AJAX in the reducer.

As a result, with a history of actions, you can always re-generate the same state. If you remove an action from the history, you can generate a different state. Logux uses it for reverting changes and edits conflict resolution.

By default, Logux keeps last 1000 action (you can change it, see [reasons chapter]) and cache state every 50 actions to make time-travel faster.

[reasons chapter]: ./6-reason.md

## Client State and UI

Logux Redux generates big JS object as application state. We recommend to use some reactive UI library (React, Vue.js, Svelte, etc.) to render and change UI according to state changes:

```js
import { useSelector } from 'react-redux'

export const Users = ({ id }) => {
  const users = useSelector(state => state.users)
  return <>
    {users.map(user => <User key={user.id} user={user}>)}
  </>
}
```

Often we need non-pure logic, and we can’t put it to a reducer. For instance, we need to change `document.title` on new error. For these cases, you need to set a listener for state changes:

```js
store.subscribe(() => {
  if (store.getState().errors.length) {
    document.title = '* Error'
  } else {
    document.title = 'OK'
  }
})
```


## Server State

By default, the server state is opposite to client state. Because server-side cache could be very big, the database is the single source of truth. You can use any database with Logux.

Logux Server removes action after processing and always look to a database for the latest value. As a result, you can’t undo actions on the server.

However, you can change this behavior and have event sourcing on the server too.

Every time when client subscribes to some data, server go to database and send initial value:

<details open><summary><b>Node.js</b></summary>

```js
server.channel('users/:id', {
  …,
  async init (ctx, action, meta) {
    let user = await db.loadUser(action.userId)
    ctx.sendBack({ type: 'user/add', user })
  }
})
```

</details>
<details><summary><b>Ruby on Rails</b></summary>

```ruby
# app/logux/channels/users.rb
module Channels
  class Users < Logux::ChannelController
    def initial_data
      user = User.find(action.channel.split('/')[1])
      [{ type: 'user/add', user: user }]
    end
  end
end
```

</details>

After initial subscribing, server will just re-send changes without going to database:

<details open><summary><b>Node.js</b></summary>

```js
server.type('users/add', {
  …,
  resend (ctx, action, meta) {
    return { channel: `users/${ action.userId }` }
  },
  …
})
```

</details>
<details><summary><b>Ruby on Rails</b></summary>

*Under construction. Until `resend` will be implemented in the gem.*

</details>

Every time when user sends action, server change the data in database:

<details open><summary><b>Node.js</b></summary>

```js
server.type('users/add', {
  …,
  async process (ctx, action, meta) {
    await db.insertUser(action.user)
  }
})
```

</details>
<details><summary><b>Ruby on Rails</b></summary>

```ruby
# app/logux/actions/users.rb
module Channels
  class Users < Logux::ChannelController
    def add
      user = User.new(user_params)
      user.save!
    end

    private

    def user_params
      action.require(:user).permit(:id, :name)
    end
  end
end
```

</details>


## Conflict Resolution

If several users can work on the same document in your application, you need to think about conflict resolution. It is especially crucial for the offline-first application. However, because of network latency, online-only collaborative applications need conflict resolution too.

There is no single solution for conflict resolution. It always depends on data type and business processes. Logux gives you few basements to not care about conflict resolutions in simple cases and write custom logic in complicated situations.

**Logux Redux** uses time travel to keep the same order of actions on all machines. It uses [ID and time] from meta to detect order and time travel to insert action in the right moment of history. Time travel is a technique when Logux Redux reverts recent actions, apply new action, and then re-apply recent actions.

As a result, if the developer used [atomic actions], conflict actions will override each other (“the last write wins” model). In complicated cases, you can define merge logic in reducers.

By default, **Server** doesn’t use time travel, because the average state can’t be stored in the memory. You need manually compare action’s time and latest change time to implement “last write wins.” Similarly, you can implement any other conflict resolution logic.

<details open><summary><b>Node.js</b></summary>

```js
server.type('users/rename', {
  …,
  async process (ctx, action, meta) {
    const user = await db.getUser(action.userId)
    if (isFirstOlder(user.nameChangedAt, meta)) {
      user.name = action.name
      user.nameChangedAt = meta
      await user.save()
    }
  }
})
```

</details>
<details><summary><b>Ruby on Rails</b></summary>

```ruby
# app/logux/actions/users.rb
module Channels
  class Users < Logux::ChannelController
    def rename
      user = User.find(action[:userId])
      if user.name_changed_at <= meta.time
        user.name = action[:name]
        user.name_changed_at = meta.time
        user.save!
      end
    end
  end
end
```

</details>

[atomic actions]: ./2-action.md#atomic-actions
[ID and time]: ./3-meta.md#id-and-time


## Reverting Changes

Optimistic UI is excellent to improve your website performance. Instead of showing loader during saving, you allow a user to move to the next task. However, you need to think about two cases: offline and server error.

Offline support is easy in Logux. Logux will keep unsaved actions in the log and synchronize them later.

But processing server error is more complicated. Another reducer can already change the data according to that document was saved, but after a few minutes server can send that user doesn’t have access to change this document.

Logux Redux uses event sourcing to deal with this extreme case. In any moment server can send `logux/undo` action and client will revert changes of this action. Logux Redux will load the closest state snapshot and then re-apply all actions without reverted action.

Logux servers send `logux/undo` in 3 cases:

1. Access check didn’t pass.
2. Error during action processing.
3. A developer wrote custom code to add `logux/undo` action.

<details open><summary><b>Node.js</b></summary>

```js
server.undo(meta, 'too late')
```

</details>
<details><summary><b>Ruby on Rails</b></summary>

```ruby
Logux.undo(meta, reason: 'too late')
```

</details>

**[Next chapter →](./5-subscription.md)**
