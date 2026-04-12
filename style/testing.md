# Testing
<!-- 100 chars ------------------------------------------------------------------------------------>

The following is how I like to write unit tests in Go. It is not intended to be authoritative or
standard, and may conflict with other style guides. Instead it is a mix of personal taste and
learnings from developing and maintaining unit tests over a number of years.

## Libraries

In addition to the standard library's [`testing`] package, [`testify`] is recommended. Primarily to
improve ergonomics for assertions. For example:

```go
// with `testing.T`
if want != got {
  t.Logf("unexpected value: want: %v, go: %v", want, got)
}

// with `testify/assert`
assert.Equal(t, want, got)
```

The helpers provided by [`testify/require`] should only be used when a the remainder of the test is
predicated on a particular result. Otherwise, [`testify/assert`] should be used. For example:

```go
// `example` is required later in the test, require no error was returned
example, err := mypkg.NewExample()
require.NoError(t, err)

// no additional assertions are required, use assert instead
got, err := example.DoThing()

assert.Equal(t, want, got)
assert.NoError(t, err)
```

The `testify` package has grown somewhat organically over time, with some parts of its API no longer
being considered 'best practice'. To avoid these soft-deprecated helpers, use the [testifylint]
linter.

For further discussion on testify usage, see the [Structure] and [Mocking] sections below.

[`testing`]: https://pkg.go.dev/testing
[`testify`]: https://pkg.go.dev/github.com/stretchr/testify
[`testify/assert`]: https://pkg.go.dev/github.com/stretchr/testify/assert
[`testify/require`]: https://pkg.go.dev/github.com/stretchr/testify/require
[testifylint]: https://github.com/Antonboom/testifylint
[Structure]: #structure
[Mocking]: #mocking

## Package Names

Whenever possible, test the public functions and methods of a package only. This can be enforced by
adding `_test` suffix to the package name of `*_test.go` files. For example:

```go
// example_test.go

package example_test // append a "_test" suffix to the real package name

import (
  "example.com/example" // requires importing the package under test
)

// write tests here
```

This approach is best combined with [dependency injection], where behaviour changes are provided by
values passed to public functions and structs. It also prevents [monkey patching] of private struct
fields and other values. This tends to lead to more well-designed code and less brittle or
unrepresentative tests. It can also let unit tests act as basic usage examples to consumers of the
package.

It should be possible to exercise all code in a package via public functions and methods. However,
in a small minority of cases, this is impractical, such as when validating a sizeable range of
inputs. In this cases, it is acceptable to test these private members directly. However, it is
recommended to limit this to [pure functions] to avoid complexity.

When testing private functions or methods, the package name should match that of the code under
test. But the filename should have an `_internal_test.go` suffix to indicate the unusual nature of
the file. For example:

```go
// example_internal_test.go

package example // no "_test" suffix

// no import of the package under test required

// write tests here
```

This package and file naming convention can be enforced using the [testpackage] linter.

[dependency injection]: https://en.wikipedia.org/wiki/Dependency_injection
[monkey patching]: https://en.wikipedia.org/wiki/Monkey_patch
[pure functions]: https://en.wikipedia.org/wiki/Pure_function
[testpackage]: https://github.com/maratori/testpackage

## Test Names

Tests should be named in the following format:

- `Test_{Function}_{Aspect}` when unit testing a function.
- `Test_{Struct}_{Method}_{Aspect}` when unit testing a method of a struct.

This format is based on the naming convention of testable [examples][5], but adds an underscore
(`_`) between the required `Test` prefix and identifier. This is to allow easy differentiation of
tests for public and private functions or types.

The "aspect" of the test name should be a short, human-readable summary of what the test covers. It
will ideally be a single word. For example, "Success" and "Error" can be used happy-path and
negative test cases respectively.

[5]: https://go.dev/blog/examples#example-function-names

## Structure

Each test should be split into 3 sections:

1. Arrange: Setup everything required for the test.
2. Act: Execute the target behaviour.
3. Assert: Validate that the execution returned the expected results.

The Act section should almost always be a single statement, such as a function or method call.

For example:

```go
// arrange
example, err := mypkg.NewExample()
require.NoError(t, err)

// act
got, err := example.DoThing()

// assert
assert.Equal(t, "want", got)
assert.NoError(t, err)
```

Generally [`testify/require`] helpers should be used in the Arrange section to ensure fallible setup
has completed correctly. Conversely, [`testify/assert`] should be used in the Assert section to
allow all returned values to be checked in parallel.

In rare situations, the Arrange or Assert section may be empty. If this occurs, the comment should
remain in case later additions to the test make use of it.

The Arrange section can be empty when testing a simple pure function, or when the setup is completed
wholly within a [test table] field.

The Assert section can be empty when no values are returned within the Act section. Alternatively,
the Assert section may be empty when validating a panic as this can only be handled in the Act
section. For example:

```go
// arrange

// act
assert.Panics(t, "want", func() {
  mypkg.MustExample()
})

// assert
```

[test table]: #test-tables

## Test Tables

[Table-driven tests] allow a test to be parameterised, avoiding repetitive boilerplate across
otherwise identical tests. These parameters can then be used to create a sub-test.

These tests come in 2 main formats: a slice of structs, or a map with struct values. Maps are
preferable because they randomise the order of execution, reducing the risk of unintentional
dependencies between test cases. It also prevents accidental duplication of test names. For example:

```go
func Test_Example_Success(t *testing.T) {
  tests := map[string]struct{
    // add parameters here
  }{
    "aaa": {/*parameters*/},
    "bbb": {/*parameters*/},
    "ccc": {/*parameters*/},
  }

  for name, tt := range tests {
    t.Run(name, func(t *testing.T) {
      // write test here
      // use tt.<field> for anything that varies
    })
  }
}
```

The sub-test names should be a short-summary of what the test covers, similar to a test's name. In
practice, sub-tests names may be longer due to needing more words to differentiate it from other
sub-tests.

[Table-driven tests]: https://go.dev/wiki/TableDrivenTests

### Breaking up Test Tables

It is tempting to parameterise a test to a large degree, usually to avoid repetition ([DRY]).
However, pursuing this too far can lead to complicated and hard to maintain tests. To avoid this,
conditionals (`if`/`else`) should never appear in a sub-test. Logic variations can be created via
dependency injection of the code under test. But when conditionals start being needed, instead
create a new test with the applicable sub-tests within.

This rule can seem excessive or overly prescriptive at a glance, and this is a fair criticism.
However, it limits the complexity a sub-test can reach, and also makes tests easier to refactor when
requirements change.

[DRY]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself

## Mocking

Some libraries provide support for mocks or fakes. When such libraries exist, it is strongly
recommended to make use of them to ensure parity with the wider ecosystem. An excellent example of
this is the [`fake`] and [`interceptor`] packages that are provided by the [`controller-runtime`]
library.

When no standard mock packages exist, [`testify/mock`] and [mockery] should be used instead.
Specifically, mockery's [testify template] should be used. By default, `testify/mock` provides a
type-unsafe API that provides useful flexibility by relies on type-casting. Mockery generates
ready-to-use mocks that preserve the flexibility, but ensure callbacks are type-safe. Additionally,
mockery automates the call to `Mock.AssertExpectations` via a factory when creating the mock
instance. For example:

```go
// using a generated mock
example := mypkg.NewMockExample(t)
example.EXPECT().DoThing().Return("want", nil /*err*/).Once()
example.EXPECT().DoThing().RunAndReturn(func () error { return "want", nil}).Once()
```

[`fake`]: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client/fake
[`interceptor`]: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client/interceptor
[`controller-runtime`]: https://pkg.go.dev/sigs.k8s.io/controller-runtime
[`testify/mock`]: https://pkg.go.dev/github.com/stretchr/testify/assert
[mockery]: https://github.com/vektra/mockery
[testify template]: https://vektra.github.io/mockery/latest/template/testify/
