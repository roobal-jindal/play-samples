# akka-grpc-play-quickstart-scala

This is a starter application that shows how to use Akka gRPC to both expose and use gRPC services inside an Play application.

For detailed documentation refer to https://www.playframework.com/documentation/latest/Home and https://developer.lightbend.com/docs/akka-grpc/current/ .

## Preparing

As gRPC requires using HTTP/2 some additional steps have to be taken before you start the project.
Since the project is configured to run with HTTP/2 so we can try out the real end-to-end gRPC interaction,
you will need to change the:

```
play.http.secret.key = ...
``` 

in `application.conf` to different value, as we will be using the `runProd` mode to launch the play app.

You may also want to look at the `project/plugins.sbt` and `build.sbt` or `build.gradle` where the akka-grpc plugin is included. 

Note that the source generation for the services/client is all done by the `AkkaGrpcPlugin`, so remember to `enablePlugins(AkkaGrpcPlugin)` it when using sbt in all the projects of your build that want to use it.

## Running

Running this application requires [sbt](http://www.scala-sbt.org/). gRPC, in turn, requires the transport to be HTTP/2 so we want Play to use HTTP/2 (which, in Play, implies HTTPS). So rather than invoke `sbt` directly we'll use a helper script invoked as such:

```
./ssl-play runProd
```

This will also generate the server and client sources based on the `app/proto/*.proto` files, thanks to the Akka gRPC
plugin being enabled. 

Note that in this example repository a `generated.keystore` is prepared already, normally it would be generated for 
you in developer mode (`sbt -Dhttps.port=9443 run`), though in order to serve HTTP/2 services (such as gRPC) you (currently)
still need to run using `runProd`.

## Learning how it works

You can also have a look at the `conf/routes` file where you'll notice how one can embed an gRPC router within a normal
play application. You can in fact mix normal Play routes with gRPC routers like this to offer a mixed service like that.

### Serving (Akka) gRPC Services

You'll notice that we bind the `/` path to the `controllers.HomeController` like usual route,
and then we use the `->` router binding syntax to bind the `routers.HelloWorldRouter`. This is because gRPC services 
have paths correspond to their "methods", yet this is handled by its internal infrastructure and end-users need
not concern themselves about the exact names – clients too are generated from the appropriate `app/protobuf/helloworld.proto`
file after all.

### Injecting Akka gRPC Clients 

Next we will want to make use of an existing gRPC service.

Similarily to the server side, the sources are generated by the Akka gRPC plugin by having it configured to emit the client as well:

```
// build.sbt
akkaGrpcExtraGenerators += PlayScalaClientCodeGenerator,
``` 

In order to make the gRPC clients easily injectable, we need to enable the following module in Play as well (in this example app this has been done already though):

```
// application.conf
play.modules {
  # To enable Akka gRPC clients to be @Injected
  enabled += example.myapp.helloworld.routers.AkkaGrpcClientModule
}
```

Which in turn allows us to inject clients to any of the services defined in our `app/proto` directory, just like so:

```scala
import akka.stream.Materializer
import example.myapp.helloworld.grpc.{HelloReply, HelloRequest}
import javax.inject.Inject

import scala.concurrent.Future


class HomeController @Inject() (mat: Materializer, greeterServiceClient: GreeterServiceClient) extends InjectedController {
  implicit val ec = mat.executionContext
}
```

Since you may want to configure what service-locator or hardcoded location to use for each client, you may do so as well in
`conf/application.conf`, though we will not dive into this here.


## Verifying

Finally, since now we know what the application is: an HTTP endpoint that hits its own gRPC endpoint to reply to the incoming request. 
We can trigger such request and see it correctly reply with a "Hello Caplin!" (which is the name of a nice Capybara, google it):

```
$ curl --insecure https://localhost:9443 ; echo
Hello Caplin!
```

