- [Svelte Crossfade](#svelte-crossfade)
    - [Examples](#examples)
- [Websocket Message Router Store](#websocket-message-router-store)
  - [Message Router](#message-router)


## Svelte Crossfade

A common issue when trying to transition between two distinct layouts, animating the elements that remain betweeen both component states is that there is often layout competition issue.

This is most common when structure A is being replaced with structure B. Where A and B share a structure C. While the animation exists fading out and B in, the layout is disrupted and results in awkward offsets and abrupt jumps in element positions. 

One solution to deal with this particular problem is to use the grid layout to force the various components that will occupy the same space into the same cell.

#### Examples
* [Tab Navigation with animated crossfade div transition](https://svelte.dev/repl/efbe0f92d43a403ea2e5ac6251a55f18?version=3.29.7)

* [Notification Stack - if/else](https://svelte.dev/repl/92d5891ad4324437aa77fd4574ce4e2e?version=3.29.7)


## Websocket Message Router Store

The svelte stores provide a convienent way to map incomming websocket messages to typed stores. Below is a basic recipe that provides simple topic messaging. This can be easily extended into a more complete system. Below we setup a store that represents a websocket. When it is undefined, the assumption is that there is no available websocket resource (This could be due to an error with the websocket host). When a websocket successfully opens a connection, the handlers are attached and the store is now set the newly created websocket. This begins the chain of dependency for the rest of the data. If the connection is killed, this provides a resonable way to reconnect and re-establish all of the message handling. Because stores have an unsubscribe/subscribe behavior when the number of listeners goes from 0 -> 1 or 1 -> 0, we can shut down the resource if we don't need it active site wide.

>__Note__: If you want to make sure the socket never closes, you can simply import the websocket to the root of your application and just passively watch it to keep the unsubscribe from occuring.

```typescript
const wsUrl = WEBSOCKET //ex: ws://localhost:8080
let _websocket: WebSocket | undefined
let _message = writable<string | undefined>(undefined)
async function createManagedWebsocket() {
  return new Promise<WebSocket>((resolve, reject) => {
    if (_websocket) resolve(_websocket)
    else {
      let ws = new WebSocket(wsUrl)
      ws.onopen = (openEvent) => {
        ws.onmessage = (msgEvent: MessageEvent<string>) => {
          _message.set(msgEvent.data) //Write message into the message store. This is the kernal store for routing.
        }
        ws.onclose = (closeEvent) => {
          _websocket = undefined
          _message.set(undefined)
        }
        ws.onerror = (errorEvent) => {
          _websocket = undefined
          _message.set(undefined)
        }
        resolve(ws)
      }
    }
  })
}
```

We will only have one websocket connection open so we declare the websocket at the top level (i.e. singleton). We represent the websocket as a promise since we would like to prevent the ability to start writting mesages before connection is established. This will defer into an ```(undefined | Websocket)``` store later.

```typescript
const { subscribe } = readable(undefined, (set: (value: WebSocket | undefined) => void) => {
    const ws = createManagedWebsocket()
    ws.then(ws => {
      if (ws !== _websocket) {
        set(ws)
        _websocket = ws
      }
    }).catch(e => {
      console.error("error in opening websocket")
    })
  return function closeWebsocket() {
    _websocket?.close()
    _websocket = undefined
    _message.set(undefined)
  }
})
```

This is an optional design, but it is somewhat convient to represent writing to the websocket with an assignment operator.
```typescript
export const websocket = {
  subscribe,
  set(value: string) {
    if (_websocket)
      _websocket.send(value)
    else {
      console.warn(`Websocket was closed but a message was attempted to be sent. Message: ${value}`)
    }
  }
}
```

Now the reader part of the websocket needs to be implemented. Here the message acts as a layer of abstraction. In this custom store, the set function marshals objects into strings and passes them to the websocket. The subscribe function just delegates work to the _message store (last message received) and makes sure the websocket is turned on. This couples the subscriptions on ```message``` store to fan out uniformally to the ```_message``` and ```_websocket``` stores
```typescript
const { subscribeMessage } = _message
export const message {
  set<T>(value: T) {
    websocket.set(JSON.stringify(value)) //Json adapter that marshals any data passed into 
  },
  subscribe: (run: (value: string | undefined) => void, invalidate?: (value?: string | undefined) => void) => {
    const writableUnsubscribe = subscribeMessage(run)
    // let _ws: WebSocket | undefined
    const wsUnsubscribe = websocket.subscribe(ws => {
      // _ws = ws
      //this forwards the subscription signal to the websocket to make sure we grab a connection if we don't have one.
    })
    return () => {
      writableUnsubscribe() //make sure to disconnect dependencies when this stores shutsdown.
      wsUnsubscribe()
    }
  }
}
```

### Message Router
This is the message reader. The `RouteMap` is an interface that takes the form of _topic_:_schema_. For example:
```typescript
type HelloWorld = {
  "hello" : string
}
```
Here we are declaring that there is a topic: "_hello_" that will have string data broadcasted. This allows us to describe the routes with interfaces or type descriptions. Below is the `MessageReader` type and the factory function that creates a message reader for a given type.

```typescript
export type MessageReader<RouteMap> = {
  read: <RouteKey extends Extract<keyof RouteMap, string>> (topic: RouteKey) => Readable<RouteMap[RouteKey] | undefined>
  readWithDefault: <RouteKey extends Extract<keyof RouteMap, string>> (topic: RouteKey, value: any) => Readable<RouteMap[RouteKey]>
}
type SimpleMessage<T> = {
  topic: string,
  data: T
}

export function reader<T extends {} = any>(): MessageReader<T> {
  return {
    read<RouteKey extends Extract<keyof T, string>>(topic: RouteKey) {
      const derivied = derived(message, ($message: string | undefined, set: (x: T[RouteKey]) => void) => {
        if ($message) {
          const data: SimpleMessage<T[RouteKey]> = JSON.parse($message)
          if (data.topic === topic) {
            set(data.data)
          }
        }
      })
      return derivied
    },
    readWithDefault<RouteKey extends Extract<keyof T, string>>(topic: RouteKey, value: T[RouteKey]) {
      const derivied = derived(message, ($message: string | undefined, set: (x: T[RouteKey]) => void) => {
        if ($message) {
          const data: SimpleMessage<T[RouteKey]> = JSON.parse($message)
          if (data.topic === topic) {
            set(data.data)
          }
        }
      }, value)
      return derivied
    }
  }
}
```

In use it looks like the following in a svelte component.
```svelte
<script>
import { reader } from "./WebsocketRouter"
type HelloWorld = {
  "hello" : string
}
const reader = reader<HelloWorld>()
const hello =  reader.read("hello")
</script>
<div>Hello { $hello ?? "awaiting message" }</div>
```

We can also extend this type enforcment to our writing onto the websocket. Below is the type definition for a message writer and factory function.

```typescript
import { message } from "./Websocket"

export function send<T>(m: SimpleMessage<T>) {
  message.set(m)
}
type MessageWriter<WriteMap> = {
  write: <WriteKey extends Extract<keyof WriteMap, string>>(topic: WriteKey, data: WriteMap[WriteKey]) => void
}
export function writer<WriteMap extends {}>(): MessageWriter<WriteMap> {
  return {
    write: <WriteKey extends Extract<keyof WriteMap, string>>(topic: WriteKey, data: WriteMap[WriteKey]) => {
      send({ subject : topic, data })
    }
  }
}
```

And we can use it just like the reader.
```svelte
<script>
import { reader, writer } from "./WebsocketRouter"
type HelloWorld = {
  "hello" : string
}
const reader = reader<HelloWorld>()
const writer = writer<HelloWorld>()
const helloReader = reader.read("hello")
setTimeout(() => {
  writer.write("hello", "world")
}, 1_000)
</script>
<div>Hello { $helloReader ?? "awaiting message" }</div>
```
