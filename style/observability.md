# Observability Guide
<!-- 100 chars ------------------------------------------------------------------------------------>
High quality observability is essential to the supportability of a product. It provides insights
into the health and behaviour of the product and its component services, supporting analysis of
incidents, customer escalations, and otherwise unexpected behaviour.

## Metrics

Metrics should provide an overview of the health of a product and its component services. These
should always be based on clear goals to ensure they remain relevant, accurate and focused. These
goals will usually be answers to one of the following questions:

* How many operations occurred?
* What was the outcome of those operations?
* How long did those operations take?

An operation can be thought of as a unit of work that can itself be composed of other units of work.
They can therefore be client request, the reconciliation of a Kubernetes resource, the execution of
a Kubernetes job. These questions can be well-represented with methodologies like [Golden Signals]
or [RED] which are both good approaches for monitoring microservice architectures.

[Golden Signals]: https://sre.google/sre-book/monitoring-distributed-systems/
[RED]: https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/

### Third Party Metrics

Existing third-party metrics are preferred whenever possible, such as
[OpenTelemetry metric semantic conventions], [Controller-Runtime], [gRPC OpenTelemetry Metrics] and
[Istio Standard Metrics]. These are good at capturing application-level data, such as the behaviour
of HTTP client requests, or gRPC server responses. However, they are inadequate for capturing the
results of business logic, asynchronous workloads, and performance-critical sections of code.

OpenTelemetry metrics need to be added to a service as appropriate. For example, the following
libraries should be used in Go microservices to achieve this:

* [`otelhttp.Transport`] for HTTP clients.
* [`otelhttp.NewMiddleware`] for HTTP servers.
* [`otelgrpc.NewClientHandler`] for gRPC clients.
* [`otelgrpc.NewServerHandler`] for gRPC servers.
* [`otelsql`] for SQL clients.
* [`runtime`] for Go runtime metrics.

OpenTelemetry will often define metrics that are well-reasoned and based on feedback from a wide
community. As such, they often cover scenarios that we may not have yet encountered, but may do so
in the future. By adopting these standards as much as possible, we can benefit from the learnings of
others. As such, we should strive to adopt existing standards even when no middleware implementation
exists. For example, while `otelsql` implements [`db.client`] metrics for SQL databases, it can
also be inplemented for NoSQL databases which may lack an equivalent library.

[OpenTelemetry metric semantic conventions]: https://opentelemetry.io/docs/specs/semconv/general/metrics/
[Controller-Runtime]: https://book.kubebuilder.io/reference/metrics-reference
[gRPC OpenTelemetry Metrics]: https://github.com/grpc/proposal/blob/master/A66-otel-stats.md
[Istio Standard Metrics]: https://istio.io/latest/docs/reference/config/metrics/
[`otelhttp.Transport`]: https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp#Transport
[`otelhttp.NewMiddleware`]: https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp#NewMiddleware
[`otelgrpc.NewClientHandler`]: https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc#NewClientHandler
[`otelgrpc.NewServerHandler`]: https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc#NewServerHandler
[`otelsql`]: https://pkg.go.dev/github.com/XSAM/otelsql
[`runtime`]: https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/runtime
[`db.client`]: https://opentelemetry.io/docs/specs/semconv/db/database-metrics/

### Custom Metrics

For a simple CRUD application, custom metrics may not be required and all decisions can be observed
through the application-level metrics. When designing applications, it is worth ensuring any logic
that necessitates custom metrics is truly necessary and cannot be simplified in some way. While this
is sometimes unavoidable, adding custom metrics increases complexity and so creates a maintenance
and supportability burden.

Any custom metric must meet the following criteria:

* It offers more visibility to the behaviour of a service than third-party metrics would.
* It can represent all outcomes of the logic it covers.
* It does not contain high cardinality attributes, with ideally no more than 10 variants.

If the number of outcomes starts to conflict with the cardinality limit, consider whether they are
all necessary.

Examples of offering 'more visibility' include:

* The outcomes of individual operations within a batch API request.
* The outcome of an operation that does not have a 1-1 mapping with HTTP/gRPC status codes.
* The outcome of a Kubernetes Job or CronJob which is often otherwise unobservable.
* The current state of the system that can change independently of client interactions.

If the additional visibility required is adding 1-2 extra attributes, such as the name of the
operation, consider augmenting a third-party metric with the extra information. However, also ensure
these attributes would not violate the high cardinality rule outlined above.

To avoid conflicts with third-party metrics and improve discoverability, prefixes should be added to
new custom metrics. Service-specific metrics should be prefixed with the service that owns them. If
the metric is shared by multiple services, it should be prefixed with a project-specific identifier.

TODO: look at https://blog.olly.garden/how-to-name-your-metrics.

## Tracing

TODO: look at https://blog.olly.garden/how-to-name-your-spans.

## Logging

- Logs must be in a structured log format
- Only produce access logs in the ingress service.
- TODO: Levels
  - Internal errors must be logged at error level.
  - Client or other non-internal errors errors must not be logged at error level.
- TODO: Significant events
  - Don't produce debug logs in hot loops.
- TODO: Separate errors from messages, messages should be static strings.
- TODO: Avoid duplicating fields - add fields to logger as soon as it becomes available
- TODO: Include useful fields to support accurate filtering, e.g. rpc service/method, bucket name, etc.
- TODO: Return an error or log it, not both
- Handle errors once. Handling an error means inspecting the error value, and making a decision.
- Filename + line number must be included.
- In-scope resource identifiers must be included when available to support filtering.
- Request metadata must be included to support filter, e.g. RPC method.
- Trace ID and whether the current span is recording must be included.

## Attributes

TODO: naming conventions for attributes + log fields?

## References

- https://blog.olly.garden/how-to-name-your-metrics
- https://blog.olly.garden/how-to-name-your-spans
- https://github.com/instrumentation-score/spec
