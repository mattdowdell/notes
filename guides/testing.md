# Testing

## Go

### Race detector

### Coverage

To generate a HTML coverage report:

```sh
go test ./path/to/tests/... -coverprofile cover.out
go tool cover -html cover.out -o cover.html
```

To get the overall coverage percentage for the tests:

```sh
# TODO
```

### Benchmarks

To run all benchmarks:

```sh
go test -bench . ./path/to/tests/...
```

To include memory usage and allocations in the output:

```sh
go test -bench . -benchmem ./path/to/tests/...
```

To run benchmarks with names matching a specific substring:

```sh
go test -bench SubString -benchmem ./path/to/tests/...
```
