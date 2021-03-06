---
menu: tut
title: Streaming
weight: 305
---

## Introduction

Streaming lies at the heart of the http4s model of HTTP, in the literal sense that `EntityBody`
is just a type alias for `Stream[Task, Byte]`. Please see [entity] for details. This means
HTTP streaming is provided by both https' service support and its client support.

## Streaming responses from your service

Because `EntityBody`s are streams anyway, returning a stream as a response from your service is
simplicity itself:

```tut:book
import scala.concurrent.duration._
import fs2.time
import fs2.{Scheduler, Strategy, Task}
import org.http4s._
import org.http4s.dsl._

// fs2's `time` module needs an implicit `Scheduler`
implicit val scheduler = Scheduler.fromFixedDaemonPool(2)

// fs2 `Async` needs an implicit `Strategy`
implicit val strategy = Strategy.fromExecutionContext(scala.concurrent.ExecutionContext.Implicits.global)

// An infinite stream of the periodic elapsed time
val seconds = time.awakeEvery[Task](1.second)

val service = HttpService {
  case GET -> Root / "seconds" =>
    Ok(seconds.map(_.toString))             // `map` `toString` because there's no `EntityEncoder` for `Duration`
}
```

Streams are returned to the client as chunked HTTP responses automatically. You don't need to provide the header yourself.

For a more realistic example of streaming results from database queries to the client, please see the
[ScalaSyd 2015] example. In particular, if you want to stream JSON responses, please take note of how
it converts a stream of JSON objects to a JSON array, which is friendlier to clients.

## Consuming streams with the client

The http4s [client] supports consuming chunked HTTP responses as a stream, again because the
`EntityBody` is a stream anyway. http4s' `Client` interface consumes streams with the `streaming`
function, which takes a `Request` and a `Response => Stream[Task, A]` and returns a
`Stream[Task, A]`. Since an `EntityBody` is just a `Stream[Task, Byte]`, then, the easiest way
to consume a stream is just:

```tut:fail
client.streaming(req)(resp => resp.body)
```

That gives you a `Stream[Task, Byte]`, but you probably want something other than a `Byte`.
Here's some code intended to consume [Twitter's streaming APIs], which return a stream of JSON.

First, let's assume we want to use [Circe] for JSON support. Please see [json] for details.

```scala
libraryDependencies ++= Seq(
  "org.http4s" %% "http4s-circe" % "{{< version >}}",
  "io.circe" %% "circe-generic" % "0.6.1"
)
```

Next, we want _streaming_ JSON. Because http4s is streaming in its bones, it relies on [jawn] for
streaming JSON support. Most popular JSON libraries, including Circe, provide jawn support, as
the code below shows. What's left is to integrate jawn's streaming parsing with fs2's Stream.
That's done by [jawnfs2], which http4s' Circe module depends on transitively.

Because Twitter's Streaming APIs literally return a stream of JSON objects, _not_ a JSON array,
we want to use jawnfs2's `parseJsonStream`.

Finally, Twitter's Streaming APIs also require OAuth authentication. So our example is an OAuth
example as a bonus!

Putting it all together into a small app that will print the JSON objects forever:

```tut:book

object twstream {
  import org.http4s._
  import org.http4s.client.blaze._
  import org.http4s.client.oauth1
  import fs2.Task
  import fs2.Stream
  import fs2.io.stdout
  import fs2.text.{lines, utf8Encode}
  import jawnfs2._
  import io.circe.Json

  // jawnstreamz needs to know what JSON AST you want
  implicit val f = io.circe.jawn.CirceSupportParser.facade

  // Remember, this `Client` needs to be cleanly shutdown
  val client = PooledHttp1Client()

  /* These values are created by a Twitter developer web app.
   * OAuth signing is an effect due to generating a nonce for each `Request`.
   */
  def sign(consumerKey: String, consumerSecret: String, accessToken: String, accessSecret: String)(req: Request): Task[Request] = {
    val consumer = oauth1.Consumer(consumerKey, consumerSecret)
    val token    = oauth1.Token(accessToken, accessSecret)
    oauth1.signRequest(req, consumer, callback = None, verifier = None, token = Some(token))
  }

  /* Sign the incoming `Request`, stream the `Response`, and `parseJsonStream` the `Response`.
   * `sign` returns a `Task`, so we need to `Stream.eval` it to use a for-comprehension.
   */
  def stream(consumerKey: String, consumerSecret: String, accessToken: String, accessSecret: String)(req: Request): Stream[Task, Json] = for {
    sr  <- Stream.eval(sign(consumerKey, consumerSecret, accessToken, accessSecret)(req))
    res <- client.streaming(sr)(resp => resp.body.chunks.parseJsonStream)
  } yield res

  /* Stream the sample statuses.
   * Plug in your four Twitter API values here.
   * We map over the Circe `Json` objects to pretty-print them with `spaces2`.
   * Then we `to` them to fs2's `lines` and the to `stdout` `Sink` to print them.
   * Finally, when the stream is complete (you hit <crtl-C>), `shutdown` the `client`.
   */
  def runc = {
    val req = Request(Method.GET, Uri.uri("https://stream.twitter.com/1.1/statuses/sample.json"))
    val s   = stream("<consumerKey>", "<consumerSecret>", "<accessToken>", "<accessSecret>")(req)
    s.map(_.spaces2).through(lines).through(utf8Encode).to(stdout).onFinalize(client.shutdown).run
  }
  
  // Uncomment To Run App
  // def main(args: Array[String]): Unit = runc.unsafeRun 
}
```

```tut:book:invisible
twstream.client.shutdownNow
```

[client]: ../client
[entity]: ../entity
[ScalaSyd 2015]: https://bitbucket.org/da_terry/scalasyd-doobie-http4s
[json]: ../json
[jawn]: https://github.com/non/jawn
[jawnfs2]: https://github.com/rossabaker/jawn-fs2
[Twitter's streaming APIs]: https://dev.twitter.com/streaming/overview
[circe]: https://circe.github.io/circe/
