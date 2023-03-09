---
title: Broadcasting
---

If you're building a modern web application, you'll probably want some form of realtime communication powered by [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API).

Superflare makes it super easy to broadcast realtime [Events](/events) to your users through specific [Channels](#channels), backed by [Durable Objects](https://developers.cloudflare.com/workers/learning/using-durable-objects).

## Broadcasting Events

To broadcast realtime events to users, add a `broadcastTo` method to your [Event](/events) class, and return a Channel name:

```ts
import { Event } from "superflare";

export class PostUpdated extends Event {
  constructor(public post: Post) {
    super();
  }

  broadcastTo() {
    return `posts.${this.postId}`;
  }
}
```

This will broadcast the event to all users subscribed to the `posts.{postId}` channel:

```ts
export async function action({ params }) {
  const post = await Post.find(params.postId);

  PostUpdated.dispatch(Post);
}
```

In the browser, you can use a [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) to connect to the channel name you defined and listen for events:

```html
<script type="module">
  const channelName = "posts.1";
  const socket = new WebSocket("wss://your-app.com/channels/" + channelName);

  socket.addEventListener("message", (event) => {
    const data = JSON.parse(event.data);
    console.log(data);

    // => { event: "PostUpdated", post: { ... } }
  });
</script>
```

## Prerequisites

If you create your app with `npx superflare new`, you will be guided through the process of getting requirements in place to broadcast events out of the box.

However, if you'd like to get started with broadcasting events manually, you'll need to:

- Create a [Durable Object](https://developers.cloudflare.com/workers/learning/using-durable-objects) for your application. Superflare will guide you through this if you've created your app with `superflare new`, but you can also [copy this reference implementation](#).
- Define your Durable Object in your `wrangler.json` file.
- Add a `binding` to the [`broadcast` settings](#broadcast-settings) in your `superflare.config.ts` which corresponds with the Durable Object you defined above.
- Create a route in your framework to handle incoming WebSocket connections.
- Export the Durable Object class from your `worker.ts` entrypoint.

## Broadcast Settings

To configure broadcasting, add a `channels` key to your `superflare.config.ts` file.

The only required setting is to define a default Durable Object binding for your channels:

```ts
import { defineConfig } from "superflare";

export default defineConfig((ctx) => {
  return {
    // ...
    channels: {
      default: {
        binding: ctx.env.CHANNEL,
      },
    },
  };
});
```

You can also define additional channel bindings and authorization patterns to restrict access to specific channels:

```ts
import { defineConfig } from "superflare";
import { User } from "~/app/models/User";
import { Post } from "~/app/models/Post";

export default defineConfig((ctx) => {
  return {
    // ...
    channels: {
      default: {
        binding: ctx.env.CHANNEL,
      },
      "posts.*": {
        binding: ctx.env.POSTS_CHANNEL,
        async authorize(user: User, postId: number) {
          const post = await Post.find(postId);

          return user.id === post.userId;
        },
      },
    },
  };
});
```

## Channels

Channels are a way to group users together, and broadcast events to them. You can use channels to broadcast events to a specific user, or to a group of users.

Unique channel names correspond to unique instances of a given Durable Object class.

### Public Channels

By default, all channels are public, and any user can subscribe to them. You can restrict access to a channel by defining an `authorize` method in your `superflare.config.ts` file.

## Handling WebSocket Connections

You will need to define a route in your framework to send WebSocket connections from the browser.

Superflare provides a `handleWebSockets` method to handle the WebSocket connections as well as handling any authorization for channels you have defined within your `superflare.config.ts` file.

You must provide the `Request` as well as the `SuperflareAuth` and `SuperflareSession` instances to the helper function in order to support authorization.

```ts
// app/routes/channels.$.ts
import { handleWebSockets } from "superflare";

export async loader({request, context: { auth, session }}) {
  return handleWebSockets(request, auth, session);
}
```

Any GET requests to `/channels/{channelName}` will be handled by the WebSocket handler within your Durable Object.

Superflare assumes the last part of the Request URL pathname separated by `/` will be the name of your channel. If you'd like to customize this behavior, you can pass a custom `channelName` to the `handleWebSockets` method.

```ts
return handleWebSockets(request, auth, session, {
  channelName: "my-custom-channel",
});
```