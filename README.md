spray-socketio
==============

socket.io for spray

Example (in progressing):

```scala

package spray.contrib.socketio.examples

import akka.io.IO
import akka.actor.{ ActorSystem, Actor, Props, ActorLogging }
import akka.pattern._
import spray.can.Http
import spray.can.server.UHttp
import spray.can.websocket
import spray.can.websocket.frame.Frame
import spray.can.websocket.frame.FrameRender
import spray.can.websocket.frame.TextFrame
import spray.contrib.socketio
import spray.contrib.socketio.Namespace
import spray.contrib.socketio.Namespace.OnEvent
import spray.contrib.socketio.packet.ConnectPacket
import spray.contrib.socketio.packet.PacketParser
import spray.http.{ HttpMethods, HttpRequest, Uri, HttpResponse, HttpEntity }
import org.parboiled2.ParseError
import rx.lang.scala.Observer
import scala.concurrent.duration._
import spray.json.DefaultJsonProtocol

object SimpleServer extends App with MySslConfiguration {
  implicit val system = ActorSystem()

  class SocketIOServer extends Actor with ActorLogging {

    def receive = {
      // when a new connection comes in we register ourselves as the connection handler
      case Http.Connected(remoteAddress, localAddress) =>
        log.info("Connected to HttpListener, sender {}", sender().path)
        sender() ! Http.Register(self)

      // socket.io handshake
      case socketio.HandshakeRequest(resp) =>
        log.info("socketio handshake from sender is {}", sender().path)
        sender() ! resp

      // when a client request for upgrading to websocket comes in, we send
      // UHttp.Upgrade to upgrade to websocket pipelines with an accepting response.
      case req @ websocket.HandshakeRequest(state) =>
        state match {
          case wsFailure: websocket.HandshakeFailure => sender() ! wsFailure.response
          case wsContext: websocket.HandshakeContext =>
            log.info("websocker handshaked from sender {}", sender().path)
            val newContext = if (socketio.isSocketIOConnecting(req.uri)) {
              val connectPacket = FrameRender.render(TextFrame(ConnectPacket().render))
              wsContext.withResponse(wsContext.response.withEntity(HttpEntity(connectPacket.toArray)))
            } else {
              wsContext
            }

            sender() ! UHttp.Upgrade(websocket.pipelineStage(self, newContext), newContext)
        }

      // upgraded successfully
      case UHttp.Upgraded(wsContext) =>
        socketio.contextFor(wsContext.uri, sender()) match {
          case Some(soContext) => Namespace.namespaces ! Namespace.Connected(soContext)
          case None            =>
        }
        log.info("Http Upgraded!")

      case x @ TextFrame(payload) =>
        try {
          val packets = PacketParser(payload)
          log.info("got {}, from sender {}", packets, sender().path)
          packets foreach { Namespace.namespaces ! Namespace.OnPacket(_, sender()) }
        } catch {
          case ex: ParseError => log.error(ex, "Error in parsing packet: {}" + ex.getMessage)
        }

      case x: Frame =>
      //log.info("Got frame: {}", x)

      case HttpRequest(HttpMethods.GET, Uri.Path("/pingpingping"), _, _, _) =>
        sender() ! HttpResponse(entity = "PONG!PONG!PONG!")

      case x: HttpRequest =>
        log.info("Got http req uri = {}", x.uri.path.toString.split("/").toList)
        log.info(x.toString)

    }
  }

  // --- json protocals:
  case class Message(message: String)
  case class Now(time: String)
  object TheJsonProtocol extends DefaultJsonProtocol {
    implicit val msgFormat = jsonFormat1(Message)
    implicit val nowFormat = jsonFormat1(Now)
  }
  import spray.json._
  import TheJsonProtocol._

  val observer = Observer[OnEvent](
    (next: OnEvent) => {
      next match {
        case OnEvent("Hi!", args, context) =>
          println("observed: " + next.name + ", " + next.args)
          next.replyEvent("welcome", List(Message("Greeting from spray-socketio").toJson))
          next.replyEvent("time", List(Now((new java.util.Date).toString).toJson))
        case OnEvent("time", args, context) =>
          println("observed: " + next.name + ", " + next.args)
        case _ =>
          println("observed: " + next.name + ", " + next.args)
      }
    })
  Namespace.namespaces ! Namespace.Subscribe("testendpoint", observer)

  import system.dispatcher

  val worker = system.actorOf(Props(classOf[SocketIOServer]), "websocket")

  IO(UHttp) ! Http.Bind(worker, "localhost", 8080)

  readLine("Hit ENTER to exit ...\n")
  system.shutdown()
  system.awaitTermination()
}

```