---
layout: post
title:  "Netty Hello World"
categories: netty, scala
image: netty_logo.png
---
When you read an article about async programming in java you will definitely find at least mentions of Netty. I've heard about this library, I know that many high-performance frameworks based on Netty (Akka, Vert.x, [the full list is here][1]), but I've never tried it. So it's time to write a hello world application using Netty and scala.

## What is Netty

>Netty is an asynchronous event-driven network application framework
for rapid development of maintainable high performance protocol servers & clients.

Simply put, Netty is going to be useful if you want to write a server that handles all requests in one thread (technically not one) instead of using a separate thread for each client request. What's wrong with a thread per request approach? - when we get a high load, we need to have a lot of threads to serve all clients, but having many threads causes high memory consumption and many thread context switches (the second reason is more important due to the fact that it is an expensive operation). This approach is very popular, for example, in javascript world. Node.js http server works exactly in this way.

Netty operates with [Channels][3] which are abstractions over a socket. As you may see, data can be read from or written to them  - all these operations will be non-blocking. In addition, you can combine channels in groups and execute the same operations as for individual channels. If you're creating server application you need some of the implementations of `ServerSocketChannel`. To start working with a server channel, we have to introduce another important thing - thread pools which all the magic will be run on. There are two thread pools which should be passed to the server channel: boss and workers.
The boss one is going to accept incoming connections while the workers process these accepted connections.

## What are we going to create

As a demo application, I'm going to code simple app that will be capturing image from webcam and transfer it over websocket to the client side, then javascript will draw it on the canvas - it's cross-browser and cross-platform solution, you can watch a video even on a mobile device (without audio though).

To capture images from the camera, we'll use [webcam-capture][2]. It's written in pure java and has very simple api.

## Server Side

All the code split into two parts: server side and client side. Due to the fact that I'm learning scala, the server side will be written in scala in java-style (I'm trying to adopt canonical, functional style, but this is tough to do very fast)

{% highlight scala %}
import java.awt.Dimension
import java.io.ByteArrayOutputStream
import java.util.concurrent.Executors
import javax.imageio.ImageIO

import com.github.sarxos.webcam.Webcam
import io.netty.bootstrap.ServerBootstrap
import io.netty.buffer.Unpooled
import io.netty.channel.group.{ChannelGroup, DefaultChannelGroup}
import io.netty.channel.nio.NioEventLoopGroup
import io.netty.channel.socket.SocketChannel
import io.netty.channel.socket.nio.NioServerSocketChannel
import io.netty.channel.{ChannelHandlerContext, ChannelInitializer, SimpleChannelInboundHandler}
import io.netty.handler.codec.http.websocketx.{BinaryWebSocketFrame, WebSocketFrame, WebSocketServerProtocolHandler}
import io.netty.handler.codec.http.{HttpObjectAggregator, HttpServerCodec}
import io.netty.handler.logging.{LogLevel, LoggingHandler}
import io.netty.util.concurrent.GlobalEventExecutor

import scala.concurrent.{ExecutionContext, Future}

object Application {

  implicit val ec = ExecutionContext.fromExecutor(Executors.newSingleThreadExecutor())

  val WEBSOCKET_PATH = "/websocket"
  val FPS = 30

  def main(args: Array[String]) {

    val allChannels = new DefaultChannelGroup("all", GlobalEventExecutor.INSTANCE)

    startCapture(allChannels)
    startServer(allChannels)
  }

  def startServer(group: ChannelGroup) = {
    val bossGroup = new NioEventLoopGroup(1)
    val workerGroup = new NioEventLoopGroup
    try {
      new ServerBootstrap()
        .group(bossGroup, workerGroup)
        .channel(classOf[NioServerSocketChannel])
        .handler(new LoggingHandler(LogLevel.INFO))
        .childHandler(new ChannelInitializer[SocketChannel] {
          override def initChannel(ch: SocketChannel): Unit = {
            ch.pipeline
              .addLast(new HttpServerCodec)
              .addLast(new HttpObjectAggregator(65536))
              .addLast(new WebSocketServerProtocolHandler(WEBSOCKET_PATH, null, true))
              .addLast(new WebSocketFrameHandler(group))
          }
        }).bind(8080).sync
        .channel
        .closeFuture.sync
    } finally {
      bossGroup.shutdownGracefully()
      workerGroup.shutdownGracefully()
    }
  }

  def startCapture(group: ChannelGroup) = {
    val webcam: Webcam = Webcam.getDefault
    webcam.setViewSize(new Dimension(640, 480))
    webcam.open

    Future {
      while (true) {
        val outputStream = new ByteArrayOutputStream
        ImageIO.write(webcam.getImage, "jpg", outputStream)
        group.writeAndFlush(new BinaryWebSocketFrame(Unpooled.copiedBuffer(outputStream.toByteArray)))
        Thread.sleep(1000 / FPS)
      }
    }
  }
}

class WebSocketFrameHandler(group: ChannelGroup) extends SimpleChannelInboundHandler[WebSocketFrame] {

  override def userEventTriggered(ctx: ChannelHandlerContext, evt: scala.Any): Unit = {
    if (evt == WebSocketServerProtocolHandler.ServerHandshakeStateEvent.HANDSHAKE_COMPLETE) {
      group.add(ctx.channel())
    } else {
      super.userEventTriggered(ctx, evt)
    }
  }

  protected def channelRead0(ctx: ChannelHandlerContext, frame: WebSocketFrame) {
    // do nothing
  }
}
{% endhighlight %}


As you see, we don't have many lines of code (thanks, scala!). Let's go through the all key parts of this code.
There are two methods: the first one connects to default camera and starts capturing images (the details of this process won't be covered in this article), the second one starts a web server.
To define a server, there is a special class `ServerBootstrap` which is a `ServerChannel` but with a simplified interface for server creation. It provides builder-style approach for its configuration. Here are main configuration steps:

* `group` - takes two event loops, the firs one is for accepting incoming connections, the second one is for data processing
* `channel` - takes a type of the channel that will be created
* `childHandler` - takes a `ChannelHandler` that will initialize a channel. In our case, it is a `ChannelInitializer`. Its implementation is described below

The last step is `bind`, that binds our channel to given port and returns a `ChannelFuture`. That `ChannelFuture` is used for correct closing of the server channel.

Now let's go back to `ChannelInitializer`. The most interesting part of it is pipeline. Pipeline passes all the date through a chain of `ChannelHandler`s. In our case, we have `HttpServerCodec` which is a http protocol implementation, `WebSocketServerProtocolHandler` easily implements WebSockets (it requires `HttpObjectAggregator`) and the last one `WebSocketFrameHandler` which is our handler that does simple thing - it add newly connected client to the group of channels. This group is used in parallel thread that captures image and puts the binary into the group. Very simple!

## Client side

As per our requirements, the client application should connect to a websocket and render an image once it comes to the client side. Here is all that we need:

{% highlight html %}
<canvas id="viewport"></canvas>
<script type="text/javascript">
    var canvas = document.getElementById('viewport');
    var context = canvas.getContext('2d');

    var socket;
    if (window.WebSocket) {
        socket = new WebSocket("ws://localhost:8080/websocket");
        socket.binaryType = "arraybuffer";
        socket.onmessage = function (event) {
            draw(event.data);
        };
    } else {
        alert("Your browser does not support Web Socket.");
    }

    function draw(imgData) {
        var b64imgData = arrayBufferToBase64(imgData);
        var img = new Image();
        img.src = "data:image/jpeg;base64," + b64imgData;
        canvas.width = img.width;
        canvas.height = img.height;
        context.drawImage(img, 0, 0, 640, 480);
    }

    function arrayBufferToBase64(buffer) {
        var binary = '';
        var bytes = new Uint8Array(buffer);
        var len = bytes.byteLength;
        for (var i = 0; i < len; i++) {
            binary += String.fromCharCode(bytes[i]);
        }
        return window.btoa(binary);
    }
</script>
{% endhighlight %}

This is pure javascipt. We even don't need jQuery. The first part is standard WebSocket initialization - nothing to comment on it. There are two functions which should be described. `arrayBufferToBase64` takes [ArrayBuffer][4] (WebSocket in `socket.binaryType = "arraybuffer"` mode uses this type of object to pass binaries) and converts to Base64 string. I haven't written this function, I've stolen it from [here][5]. And the last key function here is `draw` - when we have Base64 binaries of an image we can draw it on a canvas and this is a typical code to do so.

## Demo

<p>
<img src="/assets/netty/suZVdHpAUD.gif" />
</p>

[1]: http://netty.io/wiki/related-projects.html
[2]: https://github.com/sarxos/webcam-capture
[3]: http://netty.io/4.0/api/io/netty/channel/Channel.html
[4]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
[5]:http://stackoverflow.com/questions/9267899/arraybuffer-to-base64-encoded-string
