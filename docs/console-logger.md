---
id: console-logger
title: "Console Logger"
---

## Colorful Console Logger With Log Filtering

[//]: # (TODO: make snippet type-checked using mdoc)

```scala
package zio.logging.example

import zio.logging.{ LogFilter, LogFormat, console }
import zio.{ Cause, ExitCode, LogLevel, Runtime, Scope, URIO, ZIO, ZIOAppArgs, ZIOAppDefault, ZLayer }

object ConsoleColoredApp extends ZIOAppDefault {

  override val bootstrap: ZLayer[ZIOAppArgs, Any, Any] =
    Runtime.removeDefaultLoggers >>> console(
      LogFormat.colored,
      LogFilter
        .logLevelByName(
          LogLevel.Info,
          "zio.logging.example.LivePingService" -> LogLevel.Debug
        )
        .cached
    )

  private def ping(address: String): URIO[PingService, Unit] =
    PingService
      .ping(address)
      .foldZIO(
        e => ZIO.logErrorCause(s"ping: $address - error", Cause.fail(e)),
        r => ZIO.logInfo(s"ping: $address - result: $r")
      )

  override def run: ZIO[Scope, Any, ExitCode] =
    (for {
      _ <- ping("127.0.0.1")
      _ <- ping("x8.8.8.8")
    } yield ExitCode.success).provide(LivePingService.layer)

}
```

Expected console output:

```
timestamp=2022-10-28T21:12:07.313782+02:00 level=DEBUG thread=zio-fiber-6 message="ping: /127.0.0.1"
timestamp=2022-10-28T21:12:07.326911+02:00 level=INFO thread=zio-fiber-6 message="ping: 127.0.0.1 - result: true"
timestamp=2022-10-28T21:12:07.348939+02:00 level=ERROR thread=zio-fiber-6 message="ping: x8.8.8.8 - invalid address error" cause=Exception in thread "zio-fiber-6" java.net.UnknownHostException: java.net.UnknownHostException: x8.8.8.8: nodename nor servname provided, or not known
	at java.net.Inet6AddressImpl.lookupAllHostAddr(Native Method)
	at java.net.InetAddress$PlatformNameService.lookupAllHostAddr(InetAddress.java:929)
	at java.net.InetAddress.getAddressesFromNameService(InetAddress.java:1529)
	at java.net.InetAddress$NameServiceAddresses.get(InetAddress.java:848)
	at java.net.InetAddress.getAllByName0(InetAddress.java:1519)
	at java.net.InetAddress.getAllByName(InetAddress.java:1378)
	at java.net.InetAddress.getAllByName(InetAddress.java:1306)
	at java.net.InetAddress.getByName(InetAddress.java:1256)
	at zio.logging.example.LivePingService.ping(PingService.scala:35)
	at zio.logging.example.LivePingService.ping(PingService.scala:36)
	at zio.logging.example.LivePingService.ping(PingService.scala:33)
	at zio.logging.example.ConsoleColoredApp.ping(ConsoleColoredApp.scala:37)
	at zio.logging.example.ConsoleColoredApp.run(ConsoleColoredApp.scala:45)
	at zio.logging.example.ConsoleColoredApp.run(ConsoleColoredApp.scala:46)
timestamp=2022-10-28T21:12:07.357647+02:00 level=ERROR thread=zio-fiber-6 message="ping: x8.8.8.8 - error" cause=Exception in thread "zio-fiber-" java.net.UnknownHostException: java.net.UnknownHostException: x8.8.8.8: nodename nor servname provided, or not known
```

## JSON Console Logger 

[//]: # (TODO: make snippet type-checked using mdoc)

```scala
package zio.logging.example

import zio.logging.{ LogAnnotation, LogFormat, consoleJson }
import zio.{ ExitCode, Runtime, Scope, ZIO, ZIOAppDefault, _ }

import java.util.UUID

object ConsoleJsonApp extends ZIOAppDefault {

  case class User(firstName: String, lastName: String) {
    def toJson: String = s"""{"first_name":"$firstName","last_name":"$lastName"}""".stripMargin
  }

  private val userLogAnnotation = LogAnnotation[User]("user", (_, u) => u, _.toJson)
  private val uuid              = LogAnnotation[UUID]("uuid", (_, i) => i, _.toString)

  override val bootstrap: ZLayer[ZIOAppArgs, Any, Any] =
    Runtime.removeDefaultLoggers >>> consoleJson(
      LogFormat.default + LogFormat.annotation(LogAnnotation.TraceId) +
        LogFormat.annotation(userLogAnnotation) + LogFormat.annotation(uuid)
    )

  private val uuids = List.fill(2)(UUID.randomUUID())

  override def run: ZIO[Scope, Any, ExitCode] =
    (for {
      traceId <- ZIO.succeed(UUID.randomUUID())
      _       <- ZIO.foreachPar(uuids) { uId =>
        {
          ZIO.logInfo("Starting operation") *>
            ZIO.sleep(500.millis) *>
            ZIO.logInfo("Stopping operation")
        } @@ userLogAnnotation(User("John", "Doe")) @@ uuid(uId)
      } @@ LogAnnotation.TraceId(traceId)
      _       <- ZIO.logInfo("Done")
    } yield ExitCode.success)

}
```

Expected console output:

```
{"timestamp":"2023-02-22T00:20:09.04078+01:00 ","level":"INFO","thread":"zio-fiber-6","message":"Starting operation","trace_id":"0c787f94-d8b1-40c6-bcf9-c479a8733902","user":{"first_name":"John","last_name":"Doe"},"uuid":"b997115e-c939-485f-a931-39e16ca9f786"}
{"timestamp":"2023-02-22T00:20:09.040778+01:00","level":"INFO","thread":"zio-fiber-5","message":"Starting operation","trace_id":"0c787f94-d8b1-40c6-bcf9-c479a8733902","user":{"first_name":"John","last_name":"Doe"},"uuid":"da26ff30-57de-44fa-895b-ef7864fc8e7e"}
{"timestamp":"2023-02-22T00:20:09.576845+01:00","level":"INFO","thread":"zio-fiber-6","message":"Stopping operation","trace_id":"0c787f94-d8b1-40c6-bcf9-c479a8733902","user":{"first_name":"John","last_name":"Doe"},"uuid":"b997115e-c939-485f-a931-39e16ca9f786"}
{"timestamp":"2023-02-22T00:20:09.577009+01:00","level":"INFO","thread":"zio-fiber-5","message":"Stopping operation","trace_id":"0c787f94-d8b1-40c6-bcf9-c479a8733902","user":{"first_name":"John","last_name":"Doe"},"uuid":"da26ff30-57de-44fa-895b-ef7864fc8e7e"}
{"timestamp":"2023-02-22T00:20:09.581515+01:00","level":"INFO","thread":"zio-fiber-4","message":"Done"}
```
