# ⚠ Deprecation notice ⚠
**This package is no longer in service. Use [`webext-bridge`](https://www.npmjs.com/package/webext-bridge) instead.**

# crx-bridge

Messaging in Chrome extensions made super easy. Out of the box.

## How much easy exactly?

This much

<a name="example"></a>

```javascript
// Inside devtools script

import { sendMessage } from 'crx-bridge';

// ...

button.addEventListener('click', () => {
	const res = await sendMessage('get-selection',  { ignoreCasing: true },  'content-script');
	console.log(res); // > "The brown fox is alive and well"
})
```

```javascript
// Inside content script

import { sendMessage, onMessage } from 'crx-bridge';

onMessage('get-selection', async (message) => {
  const { sender, data: { ignoreCasing } } = message;

  console.log(sender.context, sender.tabId); // > content-script  156

  const { selection } = await sendMessage('get-preferences', { sync: false }, 'background');
  return calculateSelection(data.ignoreCasing, selection);
});
```

```javascript
// Inside background script

import { onMessage } from 'crx-bridge';

onMessage('get-preferences', ({ data }) => {
	const { sync } = data;

  return loadUserPreferences(sync);
});
```

> Examples above require transpilation and/or bundling using `webpack`/`babel`/`rollup`

`crx-bridge` handles everything for you as efficiently as possible. No more `chrome.runtime.sendMessage` or `chrome.runtime.onConnect` or `chrome.runtime.connect` ....

# Contents

- [Setup](#setup)
- [API](#api)
- [Behaviour](#behaviour)
- [Security Note](#security)
- [Troubleshooting](#troubleshooting)

<a name="setup"></a>

# Setup

### Install

```bash
$ npm i crx-bridge
```

### Light it up

Just `import Bridge from 'crx-bridge'` wherever you need it and use as shown in [example above](#example)

> Even if your extension doesn't need a background page or wont be sending/receiving messages in background script.
> <br> `crx-bridge` uses background/event context as staging area for messages, therefore it **must** loaded in background/event page for it to work.
> <br> (Attempting to send message from any context will fail silently if `crx-bridge` isn't available in background page).
> <br> See [troubleshooting section](#troubleshooting) for more.

<a name="api"></a>

# API

## `Bridge.sendMessage(messageId: string, data: any, destination: string)`

Sends a message to some other part of your extension, out of the box.

Notes:

- If there is no listener on the other side an error will be thrown where `sendMessage` was called.

- Listener on the other may want to reply. Get the reply by `await`ing the returned `Promise`

- An error thrown in listener callback (in the destination context) will behave as usual, that is, bubble up, but the same error will also be thrown where `sendMessage` was called

#### `messageId`

> Required | `string`

Any `string` that both sides of your extension agree on. Could be `get-flag-count` or `getFlagCount`, as long as it's same on receiver's `onMessage` listener.

#### `data`

> Required | `any`

Any serializable value you want to pass to other side, latter can access this value by refering to `data` property of first argument to `onMessage` callback function.

#### `destination`

> Required | `string`

The actual identifier of other endpoint.
Example: `devtools` or `content-script` or `background` or `content-script@133` or `devtools@453`

`content-script`, `window` and `devtools` destinations can be suffixed with `@<tabId>` to target specific tab. Example: `devtools@351`, points to devtools panel inspecting tab with id 351.

Read `Behaviour` section to see how destinations (or endpoints) are treated.

> Note: For security reasons, if you want to receive or send messages to or from `window` context, one of your extension's content script must call `Bridge.allowWindowMessaging(<namespace: string>)` to unlock message routing. Also call `Bridge.setNamespace(<namespace: string>)` in those `window` contexts. Use same namespace string in those two calls, so `crx-bridge` knows which message belongs to which extension (in case multiple extensions are using `crx-bride` in one page)

---

## `Bridge.onMessage(messageId: string, callback: fn)`

Register one and only one listener, per messageId per context. That will be called upon `Bridge.sendMessage` from other side.

Optionally, send a response to sender by returning any value or if async a `Promise`.

#### `messageId`

> Required | `string`

Any `string` that both sides of your extension agree on. Could be `get-flag-count` or `getFlagCount`, as long as it's same in sender's `sendMessage` call.

#### `callback`

> Required | `fn`

A callback function `Bridge` should call when a message is received with same `messageId`. The callback function will be called with one argument, a `BridgeMessage` which has `sender`, `data` and `timestamp` as its properties.

Optionally, this callback can return a value or a `Promise`, resolved value will sent as reply to sender.

Read [security note](#security) before using this.

---

## `Bridge.allowWindowMessaging(namespace: string)`

> Caution: Dangerous action

Applicable to content scripts (noop if called from anywhere else)

Unlocks the transmission of messages to and from `window` (top frame of loaded page) contexts in the tab where it is called.
`crx-bridge` by default won't transmit any payload to or from `window` contexts for security reasons.
This method can be called from a content script (in top frame of tab), which opens a gateway for messages.

Once again, `window` = the top frame of any tab. That means **allowing window messaging without checking origin first** will let JavaScript loaded at `https://evil.com` talk with your extension and possibly give indirect access to things you won't want to, like `history` API. You're expected to ensure the
safety and privacy of your extension's users.

#### `namespace`

> Required | `string`

Can be a domain name reversed like `com.github.facebook.react_devtools` or any `uuid`. Call `Bridge.setNamespace` in `window` context with same value, so that `crx-bridge` knows which payload belongs to which extension (in case there are other extensions using `crx-bridge` in a tab). Make sure namespace string is unique enough to ensure no collisions happen.

---

## `Bridge.setNamespace(namespace: string)`

Applicable to scripts in top frame of loaded remote page

Sets the namespace `Bridge` should use when relaying messages to and from `window` context. In a sense, it connects the callee context to the extension which called `Bridge.allowWindowMessaging(<namespace: string>)` in it's content script with same namespace.

#### `namespace`

> Required | `string`

Can be a domain name reversed like `com.github.facebook.react_devtools` or any `uuid`. Call `Bridge.setNamespace` in `window` context with same value, so that `crx-bridge` knows which payload belongs to which extension (in case there are other extensions using `crx-bridge` in a tab). Make sure namespace string is unique enough to ensure no collisions happen.

## Extras

The following API is built on top of `Bridge.sendMessage` and `Bridge.onMessage`, basically, it's just a wrapper, the routing and security rules still apply the same way.

### `Bridge.openStream(channel: string, destination: string)`

Opens a `Stream` between caller and destination.

Returns a `Promise` which resolves with `Stream` when the destination is ready (loaded and `Bridge.onOpenStreamChannel` callback registered).
Example below illustrates a use case for `Stream`

#### `channel`

> Required | `string`

`Stream`(s) are strictly scoped `Bridge.sendMessage`(s). Scopes could be different features of your extension that need to talk to the other side, and those scopes are named using a channel id.

#### `destination`

> Required | `string`

Same as `destination` in `Bridge.sendMessage(msgId, data, destination)`

---

### `Bridge.onOpenStreamChannel(channel: string, callback: fn)`

Registers a listener for when a `Stream` opens.
Only one listener per channel per context

#### `channel`

> Required | `string`

`Stream`(s) are strictly scoped `Bridge.sendMessage`(s). Scopes could be different features of your extension that need to talk to the other side, and those scopes are named using a channel id.

#### `callback`

> Required | `fn`

Callback that should be called whenever `Stream` is opened from the other side. Callback will be called with one argument, the `Stream` object, documented below.

`Stream`(s) can be opened by a malicious webpage(s) if your extension's content script in that tab has called `Bridge.allowWindowMessaging`, if working with sensitive information use `isInternalEndpoint(stream.info.endpoint)` to check, if `false` call `stream.close()` immediately.

### Stream Example

```javascript
// background.js

// To-Do
```

<a name="behaviour"></a>

# Behaviour

> Following rules apply to `destination` being specified in `Bridge.sendMessage(msgId, data, destination)` and `Bridge.openStream(channelId, initialData, destination)`

- Specifying `devtools` as destination from `content-script` will auto-route payload to inspecting `devtools` page if open and listening.

- Specifying `content-script` as destination from `devtools` will auto-route the message to inspected window's top `content-script` page if listening. If page is loading, message will be queued up and deliverd when page is ready and listening.

- If `window` context (which could be a script injected by content script) are source or destination of any payload, transmission must be first unlocked by calling `Bridge.allowWindowMessaging(<namespace: string>)` inside that page's top content script, since `Bridge` will first deliver the payload to `content-script` using rules above, and latter will take over and forward accordingly. `content-script` <-> `window` messaging happens using `window.postMessage` API. Therefore to avoid conflicts, `Bridge` requires you to call `Bridge.setNamespace(uuidOrReverseDomain)` inside the said window script (injected or remote, doesn't matter).

- Specifying `devtools` or `content-script` or `window` from `background` will throw an error. When calling from `background`, destination must be suffixed with tab id. Like `devtools@745` for `devtools` inspecting tab id 745 or `content-script@351` for top `content-script` at tab id 351.

<a name="security"></a>

# Serious security note

The following note only applies if and only if, you will be sending/receiving messages to/from `window` contexts. There's no security concern if you will be only working with `content-script`, `background` or `devtools` scope, which is default setting.

`window` context(s) in tab `A` get unlocked the moment you call `Bridge.allowWindowMessaging(namespace)` somewhere in your extenion's content script(s) that's also loaded in tab `A`.

Unlike `chrome.runtime.sendMessage` and `chrome.runtime.connect`, which requires extension's manifest to specify sites allowed to talk with the extension, `crx-bridge` has no such measure by design, which means any webpage whether you intended or not, can do `Bridge.sendMessage(msgId, data, 'background')` or something similar that produces same effect, as long as it uses same protocol used by `crx-bridge` and namespace set to same as yours.

So to be safe, if you will be interacting with `window` contexts, treat `crx-bridge` as you would treat `window.postMessage` API.

Before you call `Bridge.allowWindowMessaging`, check if that page's `window.location.origin` is something you expect already.

As an example if you plan on having something critical, **always** verify the `sender` before responding:

```javascript
// background.js
import { onMessage, isInternalEndpoint } from 'crx-bridge';

onMessage('getUserBrowsingHistory', (message) => {
  const { data, sender } = message;
  // Respond only if request is from 'devtools', 'content-script' or 'background' endpoint
  if (isInternalEndpoint(sender)) {
    const { range } = data;
    return getHistory(range);
  }
});
```

<a name="troubleshooting"></a>

# Troubleshooting

- Doesn't work?
  <br>If `window` contexts are not part of the puzzle, `crx-bridge` works out of the box for messaging between `devtools` <-> `background` <-> `content-script`(s). If even that is not working, it's likely that `crx-bridge` hasn't been loaded in background page of your extension, which is used by `crx-bridge` as a staging area. If you don't need a background page for yourself, here's bare minimum to get `crx-bridge` going.

```javascript
// background.js (requires transpilation/bundling using webpack(recommended))

import 'crx-bridge';
```

```javascript
// manifest.json

{
	"background": {
		"scripts": ["path/to/transpiled/background.js"]
	}
}
```

- Can't send messages to `window`?
  <br>Sending or receiving messages from or to `window` requires you to open the messaging gateway in content script(s) for that particular tab. Call `Bridge.allowWindowMessaging(<namespaceA: string>)` in any of your content script(s) in that tab and call `Bridge.setNamespace(<namespaceB: string>)` in the
  script loaded in top frame i.e the `window` context. Make sure that `namespaceA === namespaceB`. If you're doing this, read the [security note above](#security)
