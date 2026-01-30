# Testing
<!-- 100 chars ------------------------------------------------------------------------------------>

The following is how I like to write unit tests in Go. It is not intended to be authoritative or
standard, and may conflict with other style guides. Instead it is a mix of personal taste and
learnings from developing and maintaining unit tests over a number of years.

## Libraries

<!--
testing (stdlib)
testify assert/require (third-party)
-->

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

This approach is best combined with [dependency injection][1], where behaviour changes are provided by
values passed to public functions and structs. It also prevents [monkey patching] of private struct
fields and other values. This tends to lead to more well-designed code and less brittle or
unrepresentative tests. It can also let unit tests act as basic usage examples to consumers of the
package.

It should be possible to exercise all code in a package via public functions and methods. However,
in a small minority of cases, this is impractical, such as when validating a sizeable range of
inputs. In this cases, it is acceptable to test these private members directly. However, it is
recommended to limit this to [pure functions][3] only to avoid complexity.

When testing private functions or methods, the package name should match that of the code under
test. But the filename should have an `_internal_test.go` suffix to indicate the unusual nature of
the file. For example:

```go
// example_internal_test.go

package example // no "_test" suffix

// no import of the package under test required

// write tests here
```

This package and file naming convention can be enforced using the [testpackage][4] linter.

[1]: https://en.wikipedia.org/wiki/Dependency_injection
[2]: https://en.wikipedia.org/wiki/Monkey_patch
[3]: https://en.wikipedia.org/wiki/Pure_function
[4]: https://github.com/maratori/testpackage

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

<!--
arrange
act
assert
-->

## Test Tables

[Table-driven tests][6] allow a test to be parameterised, avoiding repetitive boilerplate across
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

[6]: https://go.dev/wiki/TableDrivenTests

### Breaking up Test Tables

It is tempting to parameterise a test to a large degree, usually to avoid repetition ([DRY][7]).
However, pursuing this too far can lead to complicated and hard to maintain tests. To avoid this,
conditionals (`if`/`else`) should never appear in a sub-test. Logic variations can be created via
dependency injection of the code under test. But when conditionals start being needed, instead
create a new test with the applicable sub-tests within.

This rule can seem excessive or overly prescriptive at a glance, and this is a fair criticism.
However, it limits the complexity a sub-test can reach, and also makes tests easier to refactor when
requirements change.

[7]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself

## Mocking

<!--
testify/mock
mockery
-->

<!-- 100 chars ------------------------------------------------------------------------------------>
