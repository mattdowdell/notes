# Builds

## Go

### Static Binaries

Set `CGO_ENABLED=0` for fully statically linked binaries.

```sh
CGO_ENABLED=0 go build .
```

### Build Multiple Binaries

```sh
go build -o ./dist ./cmd/...
```

### Reproducible Builds

Embed a build environment agnostic path for Go files with `-trimpath`. Suppress per-build variables with `-buildid=` in `ldflags`.

```sh
go build -trimpath -ldflags="-buildid=" .
```

### Smaller Binaries

Suppress the symbol table and DWARF debug info by setting `-s` and `-w` in `-ldflags` respectively. Testing found this produced a 30% smaller binary when static linking.

```sh
go build -ldflags="-s -w" .
```

### Cross-Platform Builds

Set the `GOOS` and `GOARCH` environment variables for cross-platform binary builds without needing to resort to enumulation.

```sh
GOOS=linux GOARCH=arm64 go build .
```

## Docker

### Bind Mounts

Use bind mounts to avoid copying the whole repository into the build context. These can be combined with cache mounts to re-use Go's caches across multiple builds. The commands below assume use of the [Official Go image](https://hub.docker.com/_/golang) image on Docker Hub.

A bind mount is read only, so the built binaries must be output outside the mount. Particularly as they're used in subsequent layers and bind mounts only exist for the lifetime of a specific step.

Without vendoring:

```dockerfile
WORKDIR /go/src

RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,source=go.sum,target=go.sum \
    --mount=type=bind,source=go.mod,target=go.mod \
    go mod download -x

RUN --mount=type=bind,target=. \
    --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg \
    CGO_ENABLED=0 go build -o /go/bin/ ./cmd/...
```

With vendoring:

```dockerfile
WORKDIR /go/src

RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=bind,target=. \
    CGO_ENABLED=0 go build -o /go/bin/ ./cmd/...
```

Bind mounts may need a trailing `,z` if the builder (i.e. the docker or podman VM) is running with SELinux enabled. See [container/podman#15423] for more details. That said, podman does not notice changes to bind-mounted directories per [containers/podman#22450].

[container/podman#15423]: https://github.com/containers/podman/issues/15423
[containers/podman#22450]: https://github.com/containers/podman/issues/22450

### Image Digests

Use a specific digest for hermetic builds.

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot@sha256:8dd8d3ca2cf283383304fd45a5c9c74d5f2cd9da8d3b077d720e264880077c65 AS runtime
```

An architecture agnostic digest can be found by pulling and inspecting an image:

```sh
# docker
docker pull gcr.io/distroless/static-debian12:nonroot
docker inspect gcr.io/distroless/static-debian12:nonroot | jq -r '.[0].RepoDigests[0]' | cut -d@ -f2

# podman
# podman seems to return 2 digests, while docker just returns 1.
# you may need to confirm the docker output to get the right one as they're ordered alphabetically.
podman pull gcr.io/distroless/static-debian12:nonroot
podman inspect gcr.io/distroless/static-debian12:nonroot | jq -r '.[0].RepoDigests[]' | cut -d@ -f2
```

### Generic Runtime Target

Use build args to switch the binary present in the final image:

```dockerfile
FROM golang:1.23-bookworm AS build

# ...

FROM gcr.io/distroless/static-debian12:nonroot AS runtime

ARG SERVICE
COPY --from=build /go/bin/${SERVICE} /${SERVICE}
```

Unfortunately, the `ENTRYPOINT` doesn't know how to deal with build args. Instead, set the entrypoint when starting the image, e.g. in docker-compose, a k8s pod spec, etc.

Build the image using:

```sh
# docker
docker build --target runtime --build-arg example --tag example:latest .

# podman
podman build --target runtime --build-arg example --tag example:latest .
```

### SOURCE_DATE_EPOCH

Set the `SOURCE_DATE_EPOCH` environment variable to a Unix time during container builds to set the timestamp on each layer. See [docs](https://docs.docker.com/build/ci/github-actions/reproducible-builds/) for more details.

This value can also be passed in as a build argument to the container to be used to set timestamps of files, which further improves reproducibility. For example:

```sh
ARG SOURCE_DATE_EPOCH=0
RUN touch --date=@${SOURCE_DATE_EPOCH} ./path/to/file
```

The `--date` option for `touch` only exists in the GNU version. It is not available on MacOS which has BSD `touch`.

### Directories affect reproducibility

A file that is added to a final image using `ADD` or `COPY` can be made reproducible by using a reproducible build, setting consistent permissions, and setting the timestamp to a static value. However, if the file is added to a directory, the directory is also modified to the current time, as described [here](https://github.com/moby/moby/issues/47438).

To workaround this issue, try to put all files in the root directory of the container as this avoids the issue.

### Docker Hub Mirror

Docker Hub has relatively low limits for anonymous access. Fortunately, there are mirrors that are far more generous:

```dockerfile
# mirror.gcr.io caches of popular docker hub images, but does not add rate limiting.
# See https://cloud.google.com/artifact-registry/docs/pull-cached-dockerhub-images.

mirror.gcr.io/IMAGE:TAG
```
