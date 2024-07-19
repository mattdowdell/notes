# Builds

## Go

### Static binaries

Set `CGO_ENABLED=0` for fully statically linked binaries.

```sh
CGO_ENABLED=0 go build .
```

### Build multiple binaries

```sh
go build -o ./dist ./cmd/...
```

### Reproducible builds

Embed a build environment agnostic path for Go files with `-trimpath`. Suppress per-build variables with `-buildid=` in `ldflags`.

```sh
go build -trimpath -ldflags="-buildid" .
```

### Smaller binaries

Suppress the symbol table and DWARF debug info by setting `-s` and `-w` in `-ldflags` respectively. Testing found this produced a 30% smaller binary when static linking.

```sh
go build -ldflags="-s -w" .
```

## Docker

```dockerfile
# Use specific digests for hermetic builds
FROM gcr.io/distroless/static-debian12:nonroot@sha256:8dd8d3ca2cf283383304fd45a5c9c74d5f2cd9da8d3b077d720e264880077c65 AS runtime
```
