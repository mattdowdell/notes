# Maturity Model

The maturity model defines a set of requirements designed to standardise maintainability, supportability and operational
expectations. A service's level matches the lowest level for which all requirements have been implemented. Achievement
of each level implies that a service has reached the minimum acceptable standard for a particular set of environments.

## Level 0 (L0)

This is the default level for services and represents an immature service. L0 services can be deployed to local
development and CI environments only.

Moving from L0 to L1 implies satisfying all L1 requirements.

## Level 1 (L1)

L1 services can be deployed to any pre-production environment. This limited scope reflects that they are mostly
feature complete, but have not undergone in-depth testing.

To achieve L1 maturity, the the service must implement the following:

1. Must document the service's core logic and dependencies, along with any important design decisions and known trade-offs.
2. Must protect and limit access to any sensitive information including secrets and personal data.
3. Must implement multi-tenancy controls for user data.
4. Must output structured logs, in a JSON format for all significant events.
5. Must log configuration on start-up, with any secrets redacted.
6. Must integrate and export off-the-shelf tracing and metrics for all IO operations, e.g. OpenTelemetry [HTTP]/[RPC]/[SQL] semantic conventions.
7. Must execute all non-startup, first-party code within a tracing span.
8. Must propagate traces with HTTP and gRPC requests.
9. Must configure exporting or scraping of controller-runtime metrics for Kubernetes controllers.
10. Must configure a readiness check if the service receives requests.
11. Must configure a liveness check if the service does not receive requests, excluding Kubernetes jobs.
12. Must define a pod disruption budget that minimises unavailability for the service.
13. Must have an anti-affinity policy to ensure a service's replicas are distributed evenly across nodes.
14. Must handle panics and produce logs and metrics to support debugging.
15. Must return actionable directions in client error responses for correcting the observed issue.
16. Must redact implementation-specific errors in server error responses in favour of generic messages.
17. Must have at least 80% unit test coverage for non-generated code.

[HTTP]: https://opentelemetry.io/docs/specs/semconv/http/
[RPC]: https://opentelemetry.io/docs/specs/semconv/rpc/
[SQL]: https://opentelemetry.io/docs/specs/semconv/database/

## Level 2 (L2)

L2 services can be deployed to production. They are stable, have undergone significant testing and are so considered to
be production-ready.

To achieve L2 maturity, the service must implement all L1 requirements and:

1. Must have a test plan defined for all features handled by this service, with all non-failure test cases implemented.
2. Must implement custom metrics when off-the-shelf metrics do not sufficiently capture the service behaviour and/or responses.
3. Must augment tracing spans with operation-specific attributes and significant events.
4. Must create child tracing spans for performance critical sections of code.
5. Must document service-specific metrics and runtime configuration.
6. Must have a service dashboard to visualise both off-the-shelf and custom metrics.
7. Must define alerts for critical issues in the service.
8. Must define runbooks covering any risks associated with restarting and critical errors to look for in logs.
9. Must implement Kubernetes resource requests based on observed usage.
10. Must support hot-reloading of regularly modified configuration.
11. Must support automatic graceful restarts when non-hot-reloaded configuration is modified, excluding Kubernetes jobs, e.g. via [reloader].
12. Must be highly available with multiple replicas, unless the service is a Kubernetes operator or job.
13. Must maintain compatibility with a minimum of the next and previous production release, including APIs, metrics, runtime configuration and database schemas.

[reloader]: https://github.com/stakater/Reloader

## Level 3 (L3)

Level 3 services are mature, hardened and well-documented production-ready services. It should be possible for SRE to
manage them entirely using the service's documentation and monitoring.

To achieve L3 maturity, the service must implement all L2 requirements, and:

1. Must ensure deployments and updates are fully automated without service interruption.
2. Must implement automated rotation of secrets.
3. Must have a test plan defined for all features handled by this service, with all test cases implemented.
