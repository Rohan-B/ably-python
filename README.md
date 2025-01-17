ably-python
-----------

![.github/workflows/check.yml](https://github.com/ably/ably-python/workflows/.github/workflows/check.yml/badge.svg)
[![PyPI version](https://badge.fury.io/py/ably.svg)](https://badge.fury.io/py/ably)


## Overview

This is a Python client library for Ably. The library currently targets the [Ably 1.1 client library specification](https://ably.com/docs/client-lib-development-guide/features).

## Running example

```python
import asyncio
from ably import AblyRest

async def main():
    async with AblyRest('api:key') as ably:
        channel = ably.channels.get("channel_name")

if __name__ == "__main__":
    asyncio.run(main())
```

## Installation

### Via PyPI

The client library is available as a [PyPI](https://pypi.python.org/pypi/ably) package.

```
pip install ably
```

Or, if you need encryption features:

```
pip install 'ably[crypto]'
```

### Via GitHub

```
git clone --recurse-submodules https://github.com/ably/ably-python.git
cd ably-python
python setup.py install
```

## Breaking API Changes in Version 1.2.0

Please see our [Upgrade / Migration Guide](UPDATING.md) for notes on changes you need to make to your code to update it to use the new API
introduced by version 1.2.0.

## Usage

All examples assume a client and/or channel has been created in one of the following ways:

With closing the client manually:

```python
from ably import AblyRest

async def main(): 
    client = AblyRest('api:key')
    channel = client.channels.get('channel_name')
    await client.close()
```

When using the client as a context manager, this will ensure that client is properly closed 
while leaving the `with` block:

```python
from ably import AblyRest

async def main():
    async with AblyRest('api:key') as ably:
        channel = ably.channels.get("channel_name")
```

You can define the logging level for the whole library, and override for a
specific module:
```python
import logging
import ably

logging.getLogger('ably').setLevel(logging.WARNING)
logging.getLogger('ably.rest.auth').setLevel(logging.INFO)
```
You need to add a handler to see any output:
```python
logger = logging.getLogger('ably')
logger.addHandler(logging.StreamHandler())
```
### Publishing a message to a channel

```python
await channel.publish('event', 'message')
```

### Querying the History

```python
message_page = await channel.history() # Returns a PaginatedResult
message_page.items # List with messages from this page
message_page.has_next() # => True, indicates there is another page
next_page = await message_page.next() # Returns a next page
next_page.items # List with messages from the second page
```

### Current presence members on a channel

```python
members_page = await channel.presence.get() # Returns a PaginatedResult
members_page.items
members_page.items[0].client_id # client_id of first member present
```

### Querying the presence history

```python
presence_page = await channel.presence.history() # Returns a PaginatedResult
presence_page.items
presence_page.items[0].client_id # client_id of first member
```

### Getting the channel status

```python
channel_status = await channel.status() # Returns a ChannelDetails object
channel_status.channel_id # Channel identifier
channel_status.status # ChannelStatus object
channel_status.status.occupancy # ChannelOccupancy object
channel_status.status.occupancy.metrics # ChannelMetrics object
```

### Symmetric end-to-end encrypted payloads on a channel

When a 128 bit or 256 bit key is provided to the library, all payloads are encrypted and decrypted automatically using that key on the channel. The secret key is never transmitted to Ably and thus it is the developer's responsibility to distribute a secret key to both publishers and subscribers.

```python
key = ably.util.crypto.generate_random_key()
channel = rest.channels.get('communication', cipher={'key': key})
channel.publish(u'unencrypted', u'encrypted secret payload')
messages_page = await channel.history()
messages_page.items[0].data #=> "sensitive data"
```

### Generate a Token

Tokens are issued by Ably and are readily usable by any client to connect to Ably:

```python
token_details = await client.auth.request_token()
token_details.token # => "xVLyHw.CLchevH3hF....MDh9ZC_Q"
new_client = AblyRest(token=token_details)
await new_client.close()
```

### Generate a TokenRequest

Token requests are issued by your servers and signed using your private API key. This is the preferred method of authentication as no secrets are ever shared, and the token request can be issued to trusted clients without communicating with Ably.

```python
token_request = await client.auth.create_token_request(
    {
        'client_id': 'jim',
        'capability': {'channel1': '"*"'},
        'ttl': 3600 * 1000, # ms
    }
)
# => {"id": ...,
#     "clientId": "jim",
#     "ttl": 3600000,
#     "timestamp": ...,
#     "capability": "{\"*\":[\"*\"]}",
#     "nonce": ...,
#     "mac": ...}

new_client = AblyRest(token=token_request)
await new_client.close()
```

### Fetching your application's stats

```python
stats = await client.stats() # Returns a PaginatedResult
stats.items
await client.close()
```

### Fetching the Ably service time

```python
await client.time()
await client.close()
```

## Realtime client (beta)

We currently have a preview version of our first ever Python realtime client available for beta testing.
Currently the realtime client only supports authentication using basic auth and message subscription.
Realtime publishing, token authentication, and realtime presence are upcoming but not yet supported.
Check out the [roadmap](./roadmap.md) to see our plan for the realtime client.

### Installing the realtime client

The beta realtime client is available as a [PyPI](https://pypi.org/project/ably/2.0.0b2/) package.

```
pip install ably==2.0.0b2
```

### Using the realtime client

#### Creating a client

```python
from ably import AblyRealtime

async def main():
    client = AblyRealtime('api:key')
```

#### Get a realtime channel instance

```python
channel = client.channels.get('channel_name')
```

#### Subscribing to messages on a channel

```python

def listener(message):
    print(message.data)

# Subscribe to messages with the 'event' name
await channel.subscribe('event', listener)

# Subscribe to all messages on a channel
await channel.subscribe(listener)
```

Note that `channel.subscribe` is a coroutine function and will resolve when the channel is attached

#### Unsubscribing from messages on a channel

```python
# unsubscribe the listener from the channel
channel.unsubscribe('event', listener)

# unsubscribe all listeners from the channel
channel.unsubscribe()
```

#### Subscribe to connection state change

```python
# subscribe to 'failed' connection state
client.connection.on('failed', listener)

# subscribe to 'connected' connection state
client.connection.on('connected', listener)

# subscribe to all connection state changes
client.connection.on(listener)
```

#### Attach to a channel

```python
await channel.attach()
```

#### Detach from a channel

```python
await channel.detach()
```

#### Managing a connection

```python
# Establish a realtime connection.
# Explicitly calling connect() is unnecessary unless the autoConnect attribute of the ClientOptions object is false
await client.connect()

# Close a connection
await client.close()

# Send a ping
time_in_ms = await client.connection.ping()
```

## Resources

Visit https://ably.com/docs for a complete API reference and more examples.

## Requirements

This SDK supports Python 3.7+.

We regression-test the SDK against a selection of Python versions (which we update over time, 
but usually consists of mainstream and widely used versions). Please refer to [check.yml](.github/workflows/check.yml) 
for the set of versions that currently undergo CI testing.

## Known Limitations

Currently, this SDK only supports [Ably REST](https://ably.com/docs/rest). 
However, you can use the [MQTT adapter](https://ably.com/docs/mqtt) to implement [Ably's Realtime](https://ably.com/docs/realtime) features using Python.

See [our roadmap for this SDK](roadmap.md) for more information.

## Support, feedback and troubleshooting

Please visit https://ably.com/support for access to our knowledge base and to ask for any assistance.

You can also view the [community reported GitHub issues](https://github.com/ably/ably-python/issues).

To see what has changed in recent versions of Bundler, see the [CHANGELOG](CHANGELOG.md).

If you find any compatibility issues, please [do raise an issue](https://github.com/ably/ably-python/issues/new) in this repository or [contact Ably customer support](https://ably.com/support) for advice.

## Contributing

For guidance on how to contribute to this project, see [CONTRIBUTING.md](https://github.com/ably/ably-python/blob/main/CONTRIBUTING.md)
