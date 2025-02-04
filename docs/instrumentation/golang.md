---
id: golang
title: Go OpenTelemetry Instrumentation
description: Send events from your Go application to SigNoz

---

import Tabs from "@theme/Tabs";
import TabItem from "@theme/TabItem";
import InstrumentationFAQ from '../shared/instrumentation-faq.md'


<p align="center">

[![Book meeting](/img/docs/ZoomCTA1.png)](https://calendly.com/pranay-signoz/instrumentation-office-hrs)

</p>


### Automatically create traces/spans on HTTP requests

OpenTelemetry can help you jumpstart your way to observability by providing automatic instrumentation for HTTP requests. You have your choice of request routers in OpenTelemetry or you can use the standard HTTP handler. You should pick the mux that’s right for your framework.

#### Automatic instrumentation with request routers

**If you are using gin/gonic: **

```bash
# Add one line to your import() stanza depending upon your request router:
middleware "go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
```

and then inject OpenTelemetry middleware

```bash
router.Use(middleware.Middleware(serviceName))
```

_serviceName is found from the env variable. If this line is in main.go then it is already there_

**If you are using gorillamux: **

```bash
# Add one line to your import() stanza depending upon your request router:
middleware "go.opentelemetry.io/contrib/instrumentation/github.com/gorilla/mux/otelmux"
```

and then inject OpenTelemetry middleware

```bash
router.Use(middleware.Middleware(serviceName))
```

**If you are using echo: **

```bash
# Add one line to your import() stanza depending upon your request router:
middleware "go.opentelemetry.io/contrib/instrumentation/github.com/labstack/echo/otelecho"
```

and then inject OpenTelemetry middleware

```bash
router.Use(middleware.Middleware(serviceName))
```

### Run Command

```bash
SERVICE_NAME=<service_name> INSECURE_MODE=true OTEL_METRICS_EXPORTER=none OTEL_EXPORTER_OTLP_ENDPOINT=<IP of SigNoz backend:4317> go run main.go
```

_<service_name>_ is the name of the service

### If you don’t use a request router

```bash
import (
  "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

```

In each place where you pass an http.Handler to a ServeMux, you’ll wrap the handler function. For instance, you’ll make the following replacements:

```bash
- mux.Handle("/path", h)
+ mux.Handle("/path", otelhttp.NewHandler(h, "description of path"))
```

```bash
- mux.Handle("/path", http.HandlerFunc(f))
+ mux.Handle("/path", otelhttp.NewHandler(http.HandlerFunc(f), "description of path"))
```

In this fashion, you can ensure that every function you wrap with othttp will automatically have its metadata collected and a corresponding trace started.

### Automatically create traces/spans on gRPC server requests

Similarly, OpenTelemetry can also help you automatically instrument gRPC requests. To instrument any gRPC servers you have, add an Interceptor to the instantiation of the server.

```bash
import (
  grpcotel "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

func main() {
  [...]

	s := grpc.NewServer(
		grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
		grpc.StreamInterceptor(otelgrpc.StreamServerInterceptor()),
	)

}

```

### Instrumentation of a sample Golang application

We have included a sample gin/gonic application with `README.md` at https://github.com/SigNoz/sample-golang-app.

Feel free to use this repo to test out OpenTelemetry instrumentation and how to send telemetry data to SigNoz.

### Validate installation by checking for traces

With your application running, you can now verify that you’ve installed OpenTelemetry correctly by confirming that telemetry data is being reported to your observability backend.

To do this, you need to make sure that your application is actually generating data. Applications will generally not produce traces unless they are being interacted with, and opentelemetry will often buffer data before sending it. So it may take some amount of time and interaction before your application data begins to appear in your backend.

### Configuring to send data to SigNoz

```go
    // main.go
    package main

    import (
      "context"
      "log"
      "google.golang.org/grpc/credentials"

      "go.opentelemetry.io/otel"
      "go.opentelemetry.io/otel/attribute"
      "go.opentelemetry.io/otel/exporters/otlp/otlptrace"
      "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"

      "go.opentelemetry.io/otel/sdk/resource"
      sdktrace "go.opentelemetry.io/otel/sdk/trace"
    )

    var (
      serviceName  = os.Getenv("SERVICE_NAME")
      collectorURL = os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
      insecure     = os.Getenv("INSECURE_MODE")
    )

    func initTracer() func(context.Context) error {

      secureOption := otlptracegrpc.WithTLSCredentials(credentials.NewClientTLSFromCert(nil, ""))
      if len(insecure) > 0 {
        secureOption = otlptracegrpc.WithInsecure()
      }

      exporter, err := otlptrace.New(
        context.Background(),
        otlptracegrpc.NewClient(
          secureOption,
          otlptracegrpc.WithEndpoint(collectorURL),
        ),
      )

      if err != nil {
        log.Fatal(err)
      }
      resources, err := resource.New(
        context.Background(),
        resource.WithAttributes(
          attribute.String("service.name", serviceName),
          attribute.String("library.language", "go"),
        ),
      )
      if err != nil {
        log.Printf("Could not set resources: ", err)
      }

      otel.SetTracerProvider(
        sdktrace.NewTracerProvider(
          sdktrace.WithSampler(sdktrace.AlwaysSample()),
          sdktrace.WithBatcher(exporter),
          sdktrace.WithResource(resources),
        ),
      )
      return exporter.Shutdown
    }

    func main() {

      cleanup := initTracer()
      defer cleanup(context.Background())

      // rest of initialization, including creating HTTP and gRPC servers/handlers...
    }
```


#### Library and framework support

Besides OpenTelemetry core modules, it is important to install instrumentation packages for every important library and framework which your service depends upon. Beyond the critical telemetry data these components emit, library and framework integrations are often required to ensure that the trace context is properly propagated.

OpenTelemetry automatically provides instrumentation for a large number of libraries and frameworks, right out of the box.

The full list of supported instrumentation can be found in the [README](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/master/instrumentation).



<p>&nbsp;</p>

<InstrumentationFAQ />