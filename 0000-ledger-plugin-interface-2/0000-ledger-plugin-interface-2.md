---
title: The Javascript Ledger Plugin Interface Version 2
draft: 1
---
# Javascript Ledger Plugin Interface Version 2

The Interledger Protocol is a protocol suite for making payments across multiple different settlement systems.

This spec defines a JavaScript ledger abstraction interface for Interledger clients and connectors to communicate and route payments across different ledger protocols. While the exact methods and events defined here are specific to the JavaScript implementation, this may be used as a guide for ledger abstractions in other languages.

To send ILP payments through a new ledger, one must implement a ledger plugin that exposes the interface defined below. This can be used with the ILP Client and Connector and should work out of the box.

~This spec depends on the [ILP spec](../0003-interledger-protocol/).~

## Class: LedgerPlugin
`class LedgerPlugin`

###### Methods
| | Name |
|:--|:--|
| `new` | [**LedgerPlugin**](#new-ledgerplugin) ( opts ) |
| | [**connect**](#ledgerpluginconnect) ( options ) `⇒ Promise.<null>` |
| | [**disconnect**](#ledgerplugindisconnect) ( ) `⇒ Promise.<null>` |
| | [**isConnected**](#ledgerpluginisconnected) ( ) `⇒ Boolean` |
| | [**sendData**](#ledgerpluginsenddata) ( data ) <code>⇒ Promise.&lt;Buffer></code> |
| | [**sendMoney**](#ledgerpluginsendmoney) ( amount ) <code>⇒ Promise.&lt;Buffer></code> |
| | [**registerDataHandler**](#ledgerpluginregisterdatahandler) ( dataHandler ) <code>⇒ null</code> |
| | [**deregisterDataHandler**](#ledgerpluginderegisterdatahandler) ( ) <code>⇒ null</code> |
| | [**registerMoneyHandler**](#ledgerpluginregistermoneyhandler) ( moneyHandler ) <code>⇒ null</code> |
| | [**deregisterMoneyHandler**](#ledgerpluginderegistermoneyhandler) ( ) <code>⇒ null</code> |

###### Constants

| | Name |
|:--|:--|
| `static` | [**version**](#ledgerpluginversion) = 2 |

###### Events
| Name | Handler |
|:--|:--|
| [**connect**](#event-connect) | `( ) ⇒` |
| [**disconnect**](#event-disconnect) | `( ) ⇒` |
| [**error**](#event-error) | `( ) ⇒` |

### Instance Management

#### new LedgerPlugin
<code>new LedgerPlugin( **opts** : [PluginOptions](#class-pluginoptions) )</code>

Create a new instance of the plugin. Each instance typically corresponds to a different ledger. However, some plugins MAY deviate from a strict one-to-one relationship and MAY internally act as a virtual connector to more than one counterparty.

Throws `InvalidFieldsError` if the constructor is given incorrect arguments.

###### Parameters
| Name | Type | Description |
|:--|:--|:--|
| opts | <code>[PluginOptions](#class-pluginoptions)</code> | Object containing ledger-related settings. May contain plugin-specific fields. |

###### Example
```js
const ledgerPlugin = new LedgerPlugin({

  // auth parameters are defined by the plugin

  _store: {
    // persistence may be required for internal use by some ledger plugins
    // (e.g. when the ledger has reduced memo capability and we can only put an ID in the memo)
    // Store a value under a key
    put: (key, value) => {
      // Returns Promise.<null>
    },
    // Fetch a value by key
    get: (key) => {
      // Returns Promise.<Object>
    },
    // Delete a value by key
    del: (key) => {
      // Returns Promise.<null>
    }
  }
})
```

For a detailed description of these properties, please see [`PluginOptions`](#class-pluginoptions).

#### LedgerPlugin.version
<code>LedgerPlugin.**version**:Number</code>

Always `2` for this version of the Ledger Plugin Interface.

### Connection Management

#### LedgerPlugin#connect
<code>ledgerPlugin.connect( options:[ConnectOptions](#class-connectoptions ) ⇒ Promise.&lt;null></code>

`options` is optional.

Initiate ledger event subscriptions. Once `connect` is called the ledger plugin MUST attempt to subscribe to and report ledger events. Once the connection is established, the ledger plugin should emit the [`connect`](#event-connect-) event. If the connection is lost, the ledger plugin SHOULD emit the [`disconnect`](#event-disconnect-) event.

Rejects with `InvalidFieldsError` if credentials are missing, and `NotAcceptedError` if credentials are rejected.
Rejects with `TypeError` if `options.timeout` is passed but is not a `Number`.

#### LedgerPlugin#disconnect
<code>ledgerPlugin.disconnect() ⇒ Promise.&lt;null></code>

Unsubscribe from ledger events.

#### LedgerPlugin#isConnected
<code>ledgerPlugin.isConnected() ⇒ Boolean</code>

Query whether the plugin is currently connected.

#### Event: `connect`
<code>ledgerPlugin.on('connect', () ⇒ )</code>

Emitted whenever a connection is successfully established.

#### Event: `disconnect`
<code>ledgerPlugin.on('disconnect', () ⇒ )</code>

Emitted when the connection has been terminated or lost.

#### Event: `error`
<code>ledgerPlugin.on('error', ( **err**:Error ) ⇒ )</code>

General event for fatal exceptions. Emitted when the plugin experienced an unexpected unrecoverable condition. Once triggered, this instance of the plugin MUST NOT be used anymore.

### Sending

#### LedgerPlugin#sendData
<code>ledgerPlugin.sendData( **data**:Buffer ) ⇒ Promise.&lt;Buffer></code>

Sends data to the counterparty of the account and returns a response asynchronously. Request data is passed in as a `Buffer` and response data is returned as a `Buffer`.

###### Parameters
| Name | Type | Description |
|:--|:--|:--|
| data | <code>Buffer</code> | Binary request data |

###### Returns
**`Promise.<Buffer>`** A promise which resolves when the response has been received.

This method MAY reject with any arbitrary JavaScript error.

###### Example
```js
const responseBuffer = await p.sendData(requestBuffer)
```

#### LedgerPlugin#sendMoney
<code>ledgerPlugin.sendMoney( **amount**:string ) ⇒ Promise.&lt;null></code>

Transfer `amount` units of money from the caller to the counterparty of the account.

All plugins MUST support amounts in a range from one to some maximum.

### Receiving

#### LedgerPlugin#registerDataHandler
<code>ledgerPlugin.registerDataHandler( **dataHandler**: ( data: Buffer ) ⇒ Promise&lt;Buffer> ) ⇒ null</code>

Set the callback which is used to handle incoming prepared data packets. The callback should expect one parameter (the data as a Buffer)) and return a promise for the resulting response data packet (as a Buffer.) If an error occurs, the callback MAY throw an exception. In general, the callback should behave as [`sendData`](#ledgerpluginsenddata) does.

If a data handler is already set, this method throws a `DataHandlerAlreadyRegisteredError`. In order to change the data handler, the old handler must first be removed via [`deregisterDataHandler`](#ledgerpluginderegisterdatahandler). This is to ensure that handlers are not overwritten by accident.

If an incoming packet is received by the plugin, but no handler is registered, the plugin SHOULD respond with an error.

#### LedgerPlugin#deregisterDataHandler
<code>ledgerPlugin.deregisterDataHandler( ) ⇒ null</code>

Removes the currently used data handler. This has the same effect as if [`registerDataHandler`](#ledgerpluginregisterdatahandler) had never been called.

If no data handler is currently set, this method does nothing.

#### LedgerPlugin#registerMoneyHandler
<code>ledgerPlugin.registerMoneyHandler( **moneyHandler**: ( amount: string ) ⇒ Promise&lt;null> ) ⇒ null</code>

Set the callback which is used to handle incoming money. The callback should expect one parameter (the amount) and return a promise. If an error occurs, the callback MAY throw an exception. In general, the callback should behave as [`sendMoney`](#ledgerpluginsendmoney) does.

If a money handler is already set, this method throws a `MoneyHandlerAlreadyRegisteredError`. In order to change the money handler, the old handler must first be removed via [`deregisterMoneyHandler`](#ledgerpluginderegistermoneyhandler). This is to ensure that handlers are not overwritten by accident.

If incoming money is received by the plugin, but no handler is registered, the plugin SHOULD return an error (and MAY return the money.)

#### LedgerPlugin#deregisterMoneyHandler
<code>ledgerPlugin.deregisterMoneyHandler( ) ⇒ null</code>

Removes the currently used money handler. This has the same effect as if [`registerMoneyHandler`](#ledgerpluginregistermoneyhandler) had never been called.

If no money handler is currently set, this method does nothing.

## Class: PluginOptions
<code>class PluginOptions</code>

Plugin options are passed in to the [`LedgerPlugin`](#class-ledgerplugin)
constructor when a plugin is being instantiated. The fields are ledger
specific. Any fields which cannot be represented as strings are preceded with
an underscore, and listed in the table below.

###### Special Fields
| Type | Name | Description |
|:--|:--|:--|
| `Object` | [_store](#pluginoptions-_store) | Persistence layer callbacks |

### Fields

#### PluginOptions#_store
<code>**_store**:Object</code>

Provides callback hooks to the host's persistence layer.

Persistence MAY be required for internal use by some ledger plugins. For this purpose hosts MAY be configured with a persistence layer.

Method names are based on the popular LevelUP/LevelDOWN packages.

###### Example
```js
{
  // Store a value under a key
  put: (key, value) => {
    // Returns Promise.<null>
  },
  // Fetch a value by key
  get: (key) => {
    // Returns Promise.<Object>
  },
  // Delete a value by key
  del: (key) => {
    // Returns Promise.<null>
  }
}
```

## Class: ConnectOptions
<code>class ConnectOptions</code>

###### Fields
| Type | Name | Description |
|:--|:--|:--|
| `Number` | [timeout](#connectoptions-timeout) | milliseconds |

### Fields

#### ConnectOptions#timeout
<code>**timeout**:Number</code>

The number of milliseconds that the plugin should spend trying to connect before giving up.

If falsy, use the plugin's default timeout.
If `Infinity`, there is no timeout.