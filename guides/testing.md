# Testing

## Go

### Race detector

### Coverage

```sh
go test ./path/to/tests/... -coverprofile cover.out
go tool cover -html cover.out -o cover.html
```
