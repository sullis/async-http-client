# Async Http Client 
[![Build](https://github.com/AsyncHttpClient/async-http-client/actions/workflows/builds.yml/badge.svg)](https://github.com/AsyncHttpClient/async-http-client/actions/workflows/builds.yml)
![Maven Central](https://img.shields.io/maven-central/v/org.asynchttpclient/async-http-client)

Follow [@AsyncHttpClient](https://twitter.com/AsyncHttpClient) on Twitter.

The AsyncHttpClient (AHC) library allows Java applications to easily execute HTTP requests and asynchronously process HTTP responses.
The library also supports the WebSocket Protocol.

It's built on top of [Netty](https://github.com/netty/netty). It's compiled with Java 11.

## Installation

Binaries are deployed on Maven Central.
Add a dependency on the main AsyncHttpClient artifact:

Maven:
```xml
<dependencies>
    <dependency>
        <groupId>org.asynchttpclient</groupId>
        <artifactId>async-http-client</artifactId>
        <version>3.0.2</version>
    </dependency>
</dependencies>
```

Gradle:
```groovy
dependencies {
    implementation 'org.asynchttpclient:async-http-client:3.0.2'
}
```

### Dsl

Import the Dsl helpers to use convenient methods to bootstrap components:

```java
import static org.asynchttpclient.Dsl.*;
```

### Client

```java
import static org.asynchttpclient.Dsl.*;

AsyncHttpClient asyncHttpClient=asyncHttpClient();
```

AsyncHttpClient instances must be closed (call the `close` method) once you're done with them, typically when shutting down your application.
If you don't, you'll experience threads hanging and resource leaks.

AsyncHttpClient instances are intended to be global resources that share the same lifecycle as the application.
Typically, AHC will usually underperform if you create a new client for each request, as it will create new threads and connection pools for each.
It's possible to create shared resources (EventLoop and Timer) beforehand and pass them to multiple client instances in the config. You'll then be responsible for closing
those shared resources.

## Configuration

Finally, you can also configure the AsyncHttpClient instance via its AsyncHttpClientConfig object:

```java
import static org.asynchttpclient.Dsl.*;

AsyncHttpClient c=asyncHttpClient(config().setProxyServer(proxyServer("127.0.0.1",38080)));
```

## HTTP

### Sending Requests

### Basics

AHC provides 2 APIs for defining requests: bound and unbound.
`AsyncHttpClient` and Dsl` provide methods for standard HTTP methods (POST, PUT, etc) but you can also pass a custom one.

```java
import org.asynchttpclient.*;

// bound
Future<Response> whenResponse=asyncHttpClient.prepareGet("http://www.example.com/").execute();

// unbound
        Request request=get("http://www.example.com/").build();
        Future<Response> whenResponse=asyncHttpClient.executeRequest(request);
```

#### Setting Request Body

Use the `setBody` method to add a body to the request.

This body can be of type:

* `java.io.File`
* `byte[]`
* `List<byte[]>`
* `String`
* `java.nio.ByteBuffer`
* `java.io.InputStream`
* `Publisher<io.netty.buffer.ByteBuf>`
* `org.asynchttpclient.request.body.generator.BodyGenerator`

`BodyGenerator` is a generic abstraction that let you create request bodies on the fly.
Have a look at `FeedableBodyGenerator` if you're looking for a way to pass requests chunks on the fly.

#### Multipart

Use the `addBodyPart` method to add a multipart part to the request.

This part can be of type:

* `ByteArrayPart`
* `FilePart`
* `InputStreamPart`
* `StringPart`

### Dealing with Responses

#### Blocking on the Future

`execute` methods return a `java.util.concurrent.Future`. You can simply block the calling thread to get the response.

```java
Future<Response> whenResponse=asyncHttpClient.prepareGet("http://www.example.com/").execute();
        Response response=whenResponse.get();
```

This is useful for debugging but you'll most likely hurt performance or create bugs when running such code on production.
The point of using a non blocking client is to *NOT BLOCK* the calling thread!

### Setting callbacks on the ListenableFuture

`execute` methods actually return a `org.asynchttpclient.ListenableFuture` similar to Guava's.
You can configure listeners to be notified of the Future's completion.

```java
        ListenableFuture<Response> whenResponse = ???;
        Runnable callback = () - > {
            try {
               Response response = whenResponse.get();
               System.out.println(response);
            } catch (InterruptedException | ExecutionException e) {
               e.printStackTrace();
            }
        };

        java.util.concurrent.Executor executor = ???;
        whenResponse.addListener(() - > ??? , executor);
```

If the `executor` parameter is null, callback will be executed in the IO thread.
You *MUST NEVER PERFORM BLOCKING* operations in there, typically sending another request and block on a future.

#### Using custom AsyncHandlers

`execute` methods can take an `org.asynchttpclient.AsyncHandler` to be notified on the different events, such as receiving the status, the headers and body chunks.
When you don't specify one, AHC will use a `org.asynchttpclient.AsyncCompletionHandler`;

`AsyncHandler` methods can let you abort processing early (return `AsyncHandler.State.ABORT`) and can let you return a computation result from `onCompleted` that will be used
as the Future's result.
See `AsyncCompletionHandler` implementation as an example.

The below sample just capture the response status and skips processing the response body chunks.

Note that returning `ABORT` closes the underlying connection.

```java
import static org.asynchttpclient.Dsl.*;

import org.asynchttpclient.*;
import io.netty.handler.codec.http.HttpHeaders;

Future<Integer> whenStatusCode = asyncHttpClient.prepareGet("http://www.example.com/")
        .execute(new AsyncHandler<Integer> () {
            private Integer status;
            
            @Override
            public State onStatusReceived(HttpResponseStatus responseStatus) throws Exception {
                status = responseStatus.getStatusCode();
                return State.ABORT;
            }
            
            @Override
            public State onHeadersReceived(HttpHeaders headers) throws Exception {
              return State.ABORT;
            }
            
            @Override
            public State onBodyPartReceived(HttpResponseBodyPart bodyPart) throws Exception {
                 return State.ABORT;
            }
        
            @Override
            public Integer onCompleted() throws Exception{
                return status;
            }
        
            @Override
            public void onThrowable(Throwable t) {
                t.printStackTrace();
            }
        });

        Integer statusCode = whenStatusCode.get();
```

#### Using Continuations

`ListenableFuture` has a `toCompletableFuture` method that returns a `CompletableFuture`.
Beware that canceling this `CompletableFuture` won't properly cancel the ongoing request.
There's a very good chance we'll return a `CompletionStage` instead in the next release.

```java
CompletableFuture<Response> whenResponse=asyncHttpClient
        .prepareGet("http://www.example.com/")
        .execute()
        .toCompletableFuture()
        .exceptionally(t->{ /* Something wrong happened... */  })
        .thenApply(response->{ /*  Do something with the Response */ return resp;});
        whenResponse.join(); // wait for completion
```

You may get the complete maven project for this simple demo
from [org.asynchttpclient.example](https://github.com/AsyncHttpClient/async-http-client/tree/master/example/src/main/java/org/asynchttpclient/example)

## WebSocket

Async Http Client also supports WebSocket.
You need to pass a `WebSocketUpgradeHandler` where you would register a `WebSocketListener`.

```java
WebSocket websocket = c.prepareGet("ws://demos.kaazing.com/echo")
        .execute(new WebSocketUpgradeHandler.Builder().addWebSocketListener(
                new WebSocketListener() {

                  @Override
                  public void onOpen(WebSocket websocket) {
                    websocket.sendTextFrame("...").sendTextFrame("...");
                  }

                  @Override
                  public void onClose(WebSocket websocket) {
                    // ...
                  }

                  @Override
                  public void onTextFrame(String payload, boolean finalFragment, int rsv) {
                    System.out.println(payload);
                  }

                  @Override
                  public void onError(Throwable t) {
                    t.printStackTrace();
                  }
                }).build()).get();
```

## User Group

Keep up to date on the library development by joining the Asynchronous HTTP Client discussion group

[GitHub Discussions](https://github.com/AsyncHttpClient/async-http-client/discussions)
