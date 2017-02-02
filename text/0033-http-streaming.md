- Feature Name: http-streaming
- Start Date: 2016-12-12
- RFC PR: https://github.com/ponylang/rfcs/pull/71
- Pony Issue: https://github.com/ponylang/ponyc/issues/1548

# Summary

Rewrite `package/net/http` to improve memory efficiency and support streaming in various forms.

# Motivation

The current design of the http client and server packages assumes that all payloads are relatively small and that all the responses in progress, across all sessions, can be contained in server memory at once.  This fails when the payloads are large files.  For example:

* A stereo MP3 file occupies about 1 megabyte for every minute of audio.  A 3-minute Pop song therefore requires 3 MB.  A 20 minute symphony movement requires 20 MB.  And video can be 10 times larger per minute.

* Buffering all of this data before sending even the first byte delays transmission needlessly, in addition to consuming a lot of memory.

* This problem has to be solved before the Pony http package can be used to build WebDAV or media server applications.

The HTTP protocol specification provides several ways around this problem.  One is simply to send the body data a small amount at a time, allowing TCP buffering semantics to deliver the long bytestream to the other end.  No changes to the headers are required.  The other method, for use when the source really does not know the total length of the data in advance, is Chunked Transfer Encoding, where the response body is sent in smaller pieces, each with its own length header.  No `Content-Length` header is used at all in this case.

This RFC proposes implementing both of these mechanisms in the Pony stdlib `net/http` package.

# Detailed design

There are three aspects of this change that need to be implemented in various places.  These are Backpressure, multiple Transfer Modes, and data-driven transmission.  An over-arching principle is the separation of management of body data (which can be massive) from that of the URL and header portions of messages.

## Backpressure

Backpressure is essential in media streaming. For example, MP3 data is 'consumed' in the client at a rate of about 17KB/second, and video data goes considerably faster. Back on the server, a file is being read as fast as the file system can go, perhaps in excess of 1 Gigabyte per second. The network can transfer the data at somewhere in between those speeds. To prevent RAM being consumed all along the path to buffer up stuff which has been read from the file but not yet played on the client, there has to be a way for the client to signal "stop sending until I catch up".

The HTTP client code has to have a way to do that, which causes the `TCPConnection` actor to stop reading from its socket. Copying terminology from `TCPConnection`, this is the new function `_ClientConnection.mute()`.  The underlying network code then stops sending ACK packets and eventually the server end of TCP stops sending. The "throttled" event then happens on the server side, which has to be communicated to the HTTP session actor so it will stop reading from the disk file.

Then when the client catches up, it has to reverse this whole process, eventually causing the server to start reading from the file again.  This is done with the function `_ClientConnection.unmute()`.  This mechanism works the same way for large requests going *to* the server, such as in WebDAV applications.

Without this mechanism, RAM consumption in both client and server will balloon out of control. This is particularly bad on the server which might be handling hundreds of these connections simultaneously. The approach is "leave the data on disk until you know you can get rid of it".

## Transfer Modes

The primary philosophy to be applied in both the client and the server is to get rid of data as soon as possible, passing it on to whatever the next stage in processing is.

Streaming can go in either direction, whether for uploading files in a WebDAV application, or downloading files in WebDAV or media streaming.  Luckily, the existing Pony code already has a general purpose abtraction in the `Payload` class.  The current implementation of `Payload` works like this:

1. Create an empty `Payload` and set headers
2. Put data into it with one or more `add_chunk` calls
3. Send the completed `Payload` over the `TCPConnection`
4. The receiver gets a single `apply` notification
5. Pull data out with a single call to `get_body`

This is fundamentally incompatible with streaming operations.  The redesign will make HTTP exchanges on `GET` and `POST` operations much more like all the other Pony network operations which use a *push* style.  There are two ways this will work, controlled by the new `Payload.set_length` method:

2. **StreamTransfer**.  This is used for payload bodies where the exact length is known in advance, including most WebDAV and media transfers.  It is selected by calling `Payload.set_length` with an integer bytecount.  Buffer sizes determine how much data is fed to the TCP connection at once.

3. **ChunkedTransfer**.  This is used when the payload length can not be known in advance, but is likely to be large.   It is selected by calling `Payload.set_length` with a parameter of `None`.  On the TCP link this mode can be detected because there is no `Content-Length` header at all, being replaced by the `Transfer-Encoding: chunked` header.  In addition, the message body is separated into chunks, each with its own bytecount.  As with `StreamTransfer` mode, transmission can be spread out over time with the difference that it is the original data source that determines the chunk size.

Fortunately, the `PayloadBuilder` class already knows how to parse the `ChunkedTransfer` format.

The general procedure using the new interface is:

1. Create an empty `Payload` and set headers
2. Call `Payload.set_length`
3. Submit the `Payload` for transmission
4. If body data is required, feed data into the `Payload` with `add_chunk`
3. The other end receives one 'apply' call, carrying the `Payload` with headers, and one or more `chunk` notifications for body data.
4. The sender calls `Payload.finish()`
5. The receiver gets `finished()` notification

## Changes to Payload Builder

`PayloadBuilder` contains the parser that converts an incoming stream of bytes into a `Payload` object.  This is used by both client and server.

The current `PayloadBuilder` class attempts to parse and store an entire incoming payload, with no checking as to whether the size is reasonable.  The redesign changes this so that a decision is made when all of the headers have been parsed, based on the Transfer mode that will be used:

Parsing stops just after the blank line at the end of the headers.  The payload has to be dispatched to its final destination before the body can be received so that the body data can be properly dealt with as it arrives.

## Changes to Payload creation

The `Payload` class is responsible for generating its own HTTP encoding.  Support will be added for generating both streamed and chunked transmission.

1. Add a new function `Payload.set_size()` to explicitly specify the body size in advance.  Possible parameter values are:
    * `None` => Size is unknown so use Chunked Transfer Mode.  Generate a "Transfer-Encoding: chunked" header immediately.
    * `USize` => Size is known.  Generate a "Content-Length: nnn" header immediately.  This is even possible when the response comes from a file and the file system allows for size queries.

2. Currently, exactly when a `Payload` gets sent to the other end is determined by code outside of `Payload` itself.  To support streaming this has to be inverted so that calls to `Payload.add_chunk` can drive transmission directly.  This will require changes to `_ServerConnection` and `_ClientConnection`.

2. Generate Chunked Transfer Encoding.  If chunked mode has been indicated by the `Payload.set_size()` call, data added by `Payload.add_chunk` will be immediately transmitted with in the incremental format specified for chunked transfer encoding consisting of a length in hex, CRLF, the data, and another CRLF.

4. Add a `Payload.finish()` function to indicate when all of the body has been supplied.  In `ChunkedTramsfer` mode this will cause the generation of the zero-length chunk that marks the end of the body.

4. All headers are transmitted the first time any body data needs to be sent.

5. Be able to both create and respond to TCP backpressure, meaning that the channel is unable to accept more data, in the form of 'throttled' notifications.  This needs to be communicated back to the `Handler` so it can suspend reading from its data source.  For consistency, the function names in TCPConnection could be copied for this mechanism.
    * `mute()` to pause delivery of `apply()` and `chunk()` calls
    * `unmute()` to resume delivery

Yet to be determined:

2. Should `Payload.finish()` be required even for message with no body?  For example, GET, HEAD, and OPTIONS requests, and error status responses.

## Dynamic creation of Payload Handlers

Both of the existing `_ClientConnection` and `_ServerConnection` actors will be generalized to a common `HTTPSession` interface.  An HTTP Session is the external API to the communication link between client and server.  A session can only transfer one message at a time in each direction.  The client and server each have their own ways of implementing this interface, but to application code (either in the client or in the server 'back end') this interface provides a common view of how information is passed *into* the `net/http` package.

Every active `HTTPSession` requires a `PayloadReceiveHandler` at each end.

### The PayloadReceiveHandler

This is the notification interface through which HTTP messages are delivered *to* application code.  On the server, this will be HTTP Requests (GET, HEAD, DELETE, POST, etc) sent from a client and passing to the application 'back end'.  On the client, this will be the HTTP Responses coming back from the server.  The protocol is largely symmetrical and the same interface definition is used, though what processing happens behind the interface will of course vary.

Calls to these interface methods are made in the context of the `HTTPSession` actor so most of them should be
passing data on to a processing actor.

### The Handler Factory

The TCP connections that underlie HTTP sessions get created within the `net/http` package at times that the application code can not predict.  Yet, the application code has to provide `PayloadReceiveHandler` instances for these connections as necessary. To accomplish this, the application code will need to provide a `class` that implements the `HandlerFactory` interface.

The `HandlerFactory.apply` method will be called when a new `HTTPSession` is created, giving the application a chance to create an instance of its own `PayloadReceiveHandler` associated with that session.  This happens on both client and server ends.

## Changes to the Server

Information flow into the Server is as follows:

1. `Server` listens for incoming TCP connections.

2. `RequestBuilder` is the notification class for new connections.  It creates a `ServerConnection` actor and receives all the raw data from TCP.  It uses the `PayloadBuilder` parser to assemble complete `Payload` objects which are passed off to the `ServerConnection`.

3. The `ServerConnection` actor deals with *completely formed* requests that have been parsed by the `RequestBuilder`.  This is where pipelining happens, and where requests get dispatched to the caller-provided Handler.

```
actor:  Server -> TCPConnection  -> ServerConnection  +> Back end
class:            ReqBuilder        RequestHandler ---+
data:             Payload iso       Payload iso          Payload val
```
With streaming content, dispatch to the back end Handler has to happen *before* all of the body has been received.  This has to be carefully choreographed because a `Payload` is an `iso` object and can only belong to one actor at a time, yet the `RequestBuilder` is running within the `TCPConnection` actor while the `RequestHandler` is running under the `ServerConnection` actor.  Each incoming bufferful of body data, a `ByteSeq val`, will have to be handed off to `ServerConnection`, to be passed on to the Handler.

1. The existing two Handler interfaces will be renamed.  It turns out that the issues in sending a request and a response are the same, as are the issues in receiving them.  Therefore the notification interface will be `PayloadReceiveHandler` on both ends, and the sending interface will be `HTTPSession`.  This makes the code easier to read as well.

1. `PayloadReceiveHandler.apply()` will be the way the back end is informed of a new request `Payload`.  All of the headers will be present so that the request can be dispatched to the correct back end.  Subsequent calls to a new function `PayloadReceiveHandler.chunk` will provide the body data, if any.  This stream will be terminated by a call to the new function `PayloadReceiveHandler.finished`.

2. Current implementation actually dispatches multiple requests at once to the application back end when pipelined requests are sent.  This is incorrect, and is incompatible with streaming operation.  Pipelining of requests is to optimize the transmission of requests over slow links (such as over satellites), not to cause simultaneous execution on the server within one session.  This will be changed so that multiple received requests are queued and passed to the back end one at a time.  If a client wants true parallel execution of requests, it should use multiple sessions (which many browsers actually do already).

Since processing of a streaming response can take a relatively long time, acting on additional requests in the meantime does nothing but use up memory since responses would have to be queued. And if the server is being used to stream media, it is possible that these additional requests will themselves generate large responses.   Instead we will just let the requests queue up until a maximum queue length is reached (a small number) at which point back-pressure the inbound TCP stream.

## Changes to the Client
### Requests
Information flow out of the client is as follows:

1. `Client` is a single actor that manages all connections to servers.  On being presented with a request object, a `ClientConnection` object is created for the specified server host and the request `Payload` is handed off to it.

2. The `ClientConnection` actor maintains a queue of pending requests.  This is where pipelining happens, if it has been enabled by the `Client`.  Only "safe" requests (GET, HEAD, OPTIONS) should be pipelined.

```
  Client -> ClientConnection -> Payload._write -> TCP
```
This has to change to allow multiple writes to the payload body *after* it has been dispatched by `ClientConnection`.

1. `_ClientConnection` currently calls `request._write` exactly once. Instead, the `Payload` itself has to be able to control the sending of data to the server.

### Responses
Information flow back *into* the client is as follows:
```
  TCP ->         ClientConnection -> Client
  PayloadBuilder                     PayloadReceiveHandler
```

1. `PayloadReceiveHandler.chunk()` can be called more than once.  Add `PayloadReceiveHandler.closed()` to be called when all data has been received.

# How We Teach This

HTTP streaming and Chunked Transfer Encoding are parts of the HTTP standard so we do not to describe that.

Most of the stdlib packages have no external documentation beyond the automatically generated web pages that get extracted from the doc-strings. But in the source code there is sometimes quite extensive information in comments. For example, net/TCPConnection has a long discussion of how backpressure works. Some files even have little examples in them.  The largest new overall documentation for the package will come from a new package doc string.

## Examples
The `examples/httpserver` and `examples/httpget` code wlll be updated to use the new API.

Here is a skeletal version of the `httpget` example using the new API for a simple query.

```pony
actor Main
  let _env: Env
  new create(env: Env) =>
    _env = env
    try
      let client = Client(env.root as AmbientAuth)
      let factory = recover val NotifyFactory.create( _env ) end
      try
        let url = URL.build("http://host:80/path")
        let req = Payload.request("GET", url)
        client(consume req, factory)
      else
        try env.out.print("Malformed URL") end
      end
    else
      env.out.print("unable to use network")
    end

class NotifyFactory is HandlerFactory
  let _env: Env
  new iso create( env: Env ) =>
    _env = env
  fun apply( session: HTTPSession tag ): PayloadReceiveHandler iso^ =>
    HttpNotify.create( _env, session )

class HttpNotify is PayloadReceiveHandler
  """
  Handle the arrival of responses from the HTTP server.
  """
  let _env: Env
  let _session: HTTPSession tag

  new create( env': Env, session: HTTPSession tag ) =>
    _env = env'
    _session = session

  fun val apply( response: Payload val) =>
    """
    Start receiving a response.  We get the headers and maybe some body
    data.
    """
    _env.out.print(
        response.proto + " " +
        response.status.string() + " " +
        response.method)

    for (k, v) in response.headers().pairs() do
      _env.out.print(k + ": " + v)
      end

    _env.out.print("")

  fun val chunk( data: ByteSeq val ) =>
    """
    Receive additional arbitary-length response body data.
    """
    _env.out.write(piece)
    
    fun val finished() =>
     _env.out.print("-- end of body --")
    
  fun val cancelled() =>
    _env.out.print("-- response cancelled --")
    
```
# How We Test This

The existing `packages/net/http/_test.pony` does not currently test `http` operations at all.  It is an extensive test of the URL generation and parsing code however, which would not be changed by this project.

I am not sure whether the automated Pony test system could deal with two interacting programs, but:

* The program at `examples/httpget/httpget.pony` uses the *pull* model to fetch data from some server specified in the command line.  This would require a small change to use the *push* model instead.

* The program at `examples/httpserver/httpserver.pony` also uses the old interface and would have to be slightly modified.

## Testing Backpressure

The existing `examples` programs deal with very small packages of information, which is not enough to test that the TCP backpressure mechanism is being used properly.

For testing, an arbitrary amount of synthetic data could be generated, simulating reading from a file. This way the test would not require the presence of any external files, and the amount of data transfered could be changed to test out different buffering thresholds and the backpressure mechanisms. Backpressure can be tested by having the receiving end stall for a few seconds with a timer so that all the TCP buffers fill up and the backpressure notifications happen. When the timer expires and the receiver starts reading again, the reverse should happen.

# Drawbacks

1. Changes to existing clients in the way Responses are delivered, even for "simple" cases.

# Alternatives

If these changes are not done, it would remain impossible to write a serious WebDAV or media server using the `net/http` package.  (For example, something like NextCloud, currently written in PHP, or LogitechMediaServer, written in Perl.)  While the current code might work in a limited test with one user and an MP3 file, it would be quite slow and quickly run out of memory under a realistic load.

# Unresolved questions

1. Is it possible to maintain the existing *pull* interface as a layer on top of the new *push* interface?

3. What is a good *flush buffer* threshold?  Should it be tunable?
