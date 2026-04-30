# Observability Guide

High quality observability is essential to the supportability of a product. It provides insights into the health and behaviour of the product and its component services, supporting analysis of incidents, customer escalations, and otherwise unexpected behaviour.

## Metrics

Metrics should provide an overview of the health of a product and its component services. These should always be based on clear goals to ensure they remain relevant, accurate and focused. These goals will usually be answers to one of the following questions:

* How many operations occurred?
* What was the outcome of those operations?
* How long did those operations take?

An operation can be thought of as a unit of work that can itself be composed of other units of work. They can therefore be client request, the reconciliation of a Kubernetes resource, the execution of a Kubernetes job. These questions can be well-represented with methodologies like Golden Signals or RED which are both good approaches for monitoring microservice architectures.
Naming:  common prefix to all metrics produced by a given service. This enables easy metrics discovery process in grafana and is a common practice.

### Third Party Metrics

Existing third-party metrics are preferred whenever possible, such as Controller-Runtime, Istio Standard Metrics, OpenTelemetry metric semantic conventions. These are good at capturing application-level data, such as the behaviour of HTTP client requests, or gRPC server responses. However, they are inadequate for capturing the results of business logic, asynchronous workloads, and performance-critical sections of code.

Controller-Runtime metrics must be scraped from a Prometheus endpoint and will not otherwise be exported. This scraping is currently performed by opentelemetry-scraper-k8s for services such as bucketcontroller and secretmanagement.

OpenTelemetry metrics need to be added to a service as appropriate. For example, the following libraries should be used in Go microservices to achieve this:

* otelhttp.Transport for HTTP clients.
* otelhttp.NewMiddleware for HTTP servers.
* otelgrpc.NewClientHandler for gRPC clients.
* otelgrpc.NewServerHandler for gRPC servers.
* otelsql for SQL clients.

OpenTelemetry will often define metrics that are well-reasoned and based on feedback from a wide community. As such, they often cover scenarios that we may not have yet encountered, but may do so in the future. By adopting these standards as much as possible, we can benefit from the learnings of others.

_TODO: encourage adoption of standards even when no middleware implementation exists._

### Custom Metrics

For a simple CRUD application, custom metrics may not be required and all decisions can be observed through the application-level metrics. When designing applications, it is worth ensuring any logic that necessitates custom metrics is truly necessary and cannot be simplified in some way. While this is sometimes unavoidable, adding custom metrics increases complexity and so creates a maintenance and supportability burden.

Any custom metric must meet the following criteria:

* It offers more visibility to the behaviour of a service than third-party metrics would.
* It can represent all outcomes of the logic it covers.
* It does not contain high cardinality label values, with ideally no more than 10 variants.

If the number of outcomes starts to conflict with the cardinality limit, consider whether they are all necessary.

In some very rare cases, high cardinality values are unavoidable, such per-bucket usage metrics in quotamanagement. In this case, the high cardinality metrics must be derived from a separate meter provider. This separation should be paired with a separately configurable export interval derived from on the requirements specific to that data, as opposed to the default interval of 20 seconds.

Examples of offering 'more visibility' include:

* The outcomes of individual operations within a batch API request.
* The outcome of an operation that does not have a 1-1 mapping with HTTP/gRPC status codes.
* The outcome of a Kubernetes Job or CronJob which is often otherwise unobservable.
* The current state of the system that can change independently of client interactions.

If the additional visibility required is adding 1-2 extra attributes, such as the name of the operation, consider augmenting a third-party metric with the extra information. However, also ensure these attributes would not violate the high cardinality rule outlined above.

To avoid conflicts with third-party metrics, prefixes should be added to new custom metrics. Service-specific metrics should be prefixed with the service that owns them. If the metric is shared by multiple services, it should be prefixed with a project-specific identifier. Existing metrics may be renamed to match this convention, but are not required to be updated at this time.

TODO: look at https://blog.olly.garden/how-to-name-your-metrics.

## Tracing

TODO: look at https://blog.olly.garden/how-to-name-your-spans.

## Logging

TODO: Levels
TODO: Significant events
TODO: Separate errors from messages
TODO: Avoid duplicating fields - add fields to logger as soon as it becomes available
TODO: Include useful fields to support accurate filtering, e.g. rpc service/method, bucket name, etc.
TODO: Return an error or log it, not both
Handle errors once. Handling an error means inspecting the error value, and making a decision.

## Attributes

TODO: naming conventions for attributes + log fields?
