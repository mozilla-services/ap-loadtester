# Scenarios

The basic unit of a test-plan is a scenario, which indicates a set of
instructions that should be run. Many instances of a scenario can be run at once
using `aplt_testplan` to launch them.

A scenario is a Python generator function, which yields each command it wants
executed. Any additional logic the scenario wishes to perform can be done with
normal Python code. All commands are tuples, some of which return responses as
documented below.

Responses from a command can be assigned to a variable:

```python
reg = yield register(random_channel_id())
```

The response of a command can be ignored by not assigning it to a variable.

```python
yield connect()
```

Scenarios may loop forever in the event that they wish to never terminate.

**Do not call long-running functions in a scenario**, such as writing/reading
a file or making network calls. Interacting with the filesystem or network can
take time and block other scenarios from running.

See [scenarios.py](aplt/scenarios.py) for examples of scenario functions.

## Commands

Commands which send websocket messages to the server or retrieve messages from
the push service will return the Python dict of the raw server response as
documented in [the SimplePush protocol docs](http://mozilla-push-service.readthedocs.org/en/latest/design/#simplepush-protocol).

Any command that lists *arguments* must have all arguments supplied.

Full list of commands available in `aplt.commands` module:

* [connect](#connect)
* [disconnect](#disconnect)
* [hello](#hello)
* [register](#register)
* [unregister](#unregister)
* [send_notification](#send_notification)
* [expect_notification](#expect_notification)
* [ack](#ack)
* [wait](#wait)
* [timer_start](#timer_start)
* [timer_end](#timer_end)
* [counter](#counter)

Additional useful Python functions in `aplt.commands` module (these do not need
a `yield`):

* [random_channel_id](#random_channel_id)
* [random_data](#random_data)


### connect

Connects the client to the push server.

```python
yield connect()
```

**Returns:**

```python
{"messageType": "connect"}
```

### disconnect

Disconnects the client from the push server.

Note that unclean disconnects are common as many servers drop the connection
immediately.

```python
yield disconnect()
```

**Returns:**

```python
{
    "messageType": "disconnect",
    "was_clean": False,
    "code": 1006,
    "reason": "connection was closed uncleanly (server did not drop TCP connection (in time))"
}
```

### hello

Sends a hello message to the server.

**Arguments:** `None` or a user-agent ID (UAID) to use in the hello message. If
               `None` is used, the server will assign a new UAID.

```python
yield hello(None)
# or
yield hello("bf9cb2334849438293484b16c4ff2fc0")
```

**Returns:**

```python
{
    "status": 200,
    "messageType": "hello",
    "ping": 60.0,
    "uaid": "bf9cb2334849438293484b16c4ff2fc0",
    "use_webpush": True
}
```

### register

Register a channel with the push server.

**Arguments:** A channel ID.

```python
yield register("a15952b1b07d4fffa4db0318a8678105")
```

**Returns:**

```python
{
    'status': 200,
    'messageType': 'register',
    'channelID': 'a15952b1b07d4fffa4db0318a8678105',
    'pushEndpoint': 'BIG_URL'
}
```

### unregister

Remove a channel from the push server.

**Arguments:** A channel ID.

```python
yield unregister("1913165ea4104f1482ee440cedac6abd")
```

**Returns:**

```python
{
    'messageType': 'unregister',
    'status': 200,
    'channelID': '1913165ea4104f1482ee440cedac6abd',
}
```

### send_notification

Send a notification to the push service. If an empty data payload is desired,
`None` can be used for `data`.

**Arguments:** `endpoint_url`, `data`, `ttl`

```python
yield send_notification('BIG_URL', 'SOME_DATA', 60)
```

**Returns:** A tuple of (`response`, `content`) corresponding to the result of
             the notification web request. `response` is a [treq response object](http://treq.readthedocs.org/en/latest/api.html#treq.response.Response)
             object, with `content` being the response body content.

### expect_notification

Wait on the websocket connection for an expected notification to be delivered.
`time` is how long to wait before giving up.

**Arguments:** `channel_id`, `time`

```python
yield expect_notification("1913165ea4104f1482ee440cedac6abd", 5)
```

**Returns:** `None` if the timeout was hit, or the following if the expected
             notification arrived.

```python
{
    'messageType': 'notification',
    'version': 'LONG_VERSION_STRING',
    'channelID': '1913165ea4104f1482ee440cedac6abd'
}
```

### ack

Acknowledge a notification. The push server may not deliver further
notifications until sent ones are acknowledged.

**Arguments:** `channel_id`, `version`

```python
yield ack("1913165ea4104f1482ee440cedac6abd", "LONG_VERSION_STRING")
```

**Returns:** `None`

### wait

Waits a `time` seconds before proceeding.

**Arguments:** `time`

```python
yield time(10)
```

**Returns:** `None`

### timer_start

Starts a metric timer of the given `name`. An exception will be thrown if a
timer of this name was already started.

**Arguments:** `name`

```python
yield timer_start("update.latency")
```

**Returns:** `None`

### timer_end

Ends a metric timer of the given `name`. An exception will be thrown if a timer
of this name was not already started.

**Arguments:** `name`

```python
yield timer_end("update.latency")
```

**Returns:** `None`

### counter

Send a counter of the given `name` with the given `count`.

**Arguments:** `name`, `count`

```python
yield counter("notification.sent", 1)
```

### random_channel_id

Generate and return a random UUID appropriate for a UAID or channel id.

```python
channel_id = random_channel_id()
```

### random_data

Generate and return random binary data between the given min/max data length.

```python
data = random_data(2048, 4096)
```