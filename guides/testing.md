# Testing

## Go

To run tests in a specific package:

```sh
go test ./path/to/package1/ ./path/to/package2/
```

To run tests for a package and all its subpackages:

```sh
go test ./path/to/package/...
```

To run tests with names matching a specific substring:

```sh
go test -run SubString ./path/to/tests/...
```

To run a specific test:

```sh
go test -run '^TestExample$' ./path/to/package/
```

### Race detector

To enable the race detector in tests:

```sh
go test ./path/to/tests/... -race
```

Be aware that the detector is best effort. A lack of failure does not imply there are no races present, merely that no race occurred during the test execution.

### Coverage

To generate a HTML coverage report:

```sh
go test ./path/to/tests/... -coverprofile cover.out
go tool cover -html cover.out -o cover.html
```

To get the overall coverage percentage for the tests:

```sh
go test ./path/to/tests/... -coverprofile cover.out
go tool cover -func=cover.out | tail -n 1 | awk '{print $3}'
```

The `cover.out` file may include code in generated files which will rarely have coverage. These can be removed manually or automatically. If using the latter, some combination of filtering by package path and [`go/ast.IsGenerated`] is usually sufficient.

[`go/ast.IsGenerated`]: https://pkg.go.dev/go/ast#IsGenerated

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

### Integration Coverage

To generate coverage for integration tests, i.e. run tests against a running binary, the binary must first have coverage enabled at build time:

```sh
go build -cover -o ./dist/ ./cmd/example
```

A directory must then be created to hold coverage data, before starting the binary with `GOCOVERDIR` containing the path to that directory:

```sh
mkdir -p ./covdata
GOCOVERDIR=$(pwd)/covdata ./dist/example &
```

Then run the tests as normal:

```sh
go test ./path/to/integration-tests/...
```

Then skop the binary being tested to get the coverage data:

```sh
kill %1
ls -al ./covdata
```

This will have produced multiple files. These can be unified with:

```sh
go tool covdata textfmt -i=.covdata -o cover.out
```

And then processed the in the same way as unit test coverage (see above).
