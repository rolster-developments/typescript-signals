# Rolster Signals

Package containing functionality to implement messaging patterns in Typescript projects.

## Installation

```
npm i @rolster/signals
```

## Configuration

You must install the `@rolster/types` to define package data types, which are configured by adding them to the `files` property of the `tsconfig.json` file.

```json
{
  "files": ["node_modules/@rolster/types/index.d.ts"]
}
```

## Overview

The package implements two complementary messaging patterns:

- **Event Bus** — each event is handled by class-based subscribers resolved
  through dependency injection ([`@rolster/invertly`](https://www.npmjs.com/package/@rolster/invertly));
  emitting awaits every handler and returns their results. Ideal for the
  backend / command-handler side.
- **Event Broadcast** — a lightweight, synchronous pub/sub with plain function
  callbacks and per-subscription unsubscribe. Ideal for in-app fan-out
  notifications.

## Event Bus

Define a handler by extending `EventBusSubscriber<T, V>` and implementing
`execute(value)`. Register it for an event, then emit:

```typescript
import {
  EventBusSubscriber,
  registerEventBus,
  emitEventBus,
  EventBus
} from '@rolster/signals';

interface UserCreated {
  id: string;
  name: string;
}

class SendWelcomeEmail extends EventBusSubscriber<UserCreated, void> {
  execute(user: UserCreated): void {
    console.log(`Welcome, ${user.name}!`);
  }
}

// Register the subscriber for the event
registerEventBus({ event: 'user:created', subscription: SendWelcomeEmail });

// Emit it — resolves with the array of every subscriber's result
await emitEventBus({
  event: 'user:created',
  value: { id: '1', name: 'Daniel' }
});
```

You can register several subscribers for the same event; `emitEventBus` runs
them all (`Promise.all`) and returns their results. Subscriber instances are
created via `@rolster/invertly`, so they can receive injected dependencies.
When no subscriber is registered for the event, `emitEventBus` logs a warning
on the console and resolves with an empty array (`[]`).

The injectable `EventBus` class is a thin wrapper carrying a DI `context` that
is forwarded to every subscriber created for the emitted event. Decorate the
consumer with `@Injectable()` so `@rolster/invertly` can resolve it:

```typescript
import { Injectable } from '@rolster/invertly';
import { EventBus } from '@rolster/signals';

@Injectable()
class UserService {
  constructor(private readonly eventBus: EventBus) {}

  async create(name: string): Promise<void> {
    // ...persist the user...
    await this.eventBus.emit('user:created', { id: '1', name });
  }
}
```

`AbstractEventBus<K>` is the abstract contract implemented by `EventBus`
(`emit<T>(event: K, value: T): Promise<any[]>`). Use it as the injection token
when you prefer depending on the abstraction and bind it to `EventBus` (or a
test double) through the `@rolster/invertly` locator.

## Event Broadcast

A synchronous publish/subscribe. Subscribe to a channel (or to _all_ channels)
and receive an `Unsubscription` function to detach later.

```typescript
import { EventBroadcast } from '@rolster/signals';

const broadcast = new EventBroadcast<string>();

// Listen to a specific channel
const off = broadcast.subscribe<{ message: string }>('toast', (value) => {
  console.log('Toast:', value.message);
});

// Listen to every channel (receives channel + value)
broadcast.subscribeAll((channel, value) => {
  console.log(`[${channel}]`, value);
});

// Emit — reaches the channel subscribers and every "all" listener
broadcast.emit('toast', { message: 'Saved!' });

off(); // stop listening to the 'toast' channel
```

The same behaviour is available as standalone functions when you don't want to
hold an instance: `registerBroadcastChannel(channel, fn)`,
`registerBroadcastAll(fn)` and `emitBroadcast(channel, value)`.

All `EventBroadcast` instances and the standalone functions share a single,
module-level registry: a subscription made on one instance receives emissions
made from any other instance or from `emitBroadcast`. An instance is only a
typed facade over that global registry, not an isolated channel set.

When a channel has no subscribers and no "all" listener is registered,
`emitBroadcast` logs a warning on the console and does nothing else.

## Contributing

:sparkles: Daniel Andrés Castillo Pedroza
