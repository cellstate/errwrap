# errwrap

`errwrap` is a package for Go that formalizes the pattern of wrapping errors
and checking if an error contains another error.

There is a common pattern in Go of taking a returned `error` value and
then wrapping it (such as with `fmt.Errorf`) before returning it. The problem
with this pattern is that you completely lose the original `error` structure.

Arguably the _correct_ approach is that you should make a custom structure
implementing the `error` interface, and have the original error as a field
on that structure, such [as this example](http://golang.org/pkg/os/#PathError).
This is a good approach, but you have to know the entire chain of possible
rewrapping that happens, when you might just care about one.

`errwrap` formalizes this pattern (it doesn't matter what approach you use
above) by giving a single interface for wrapping errors, checking if a specific
error is wrapped, and extracting that error.

## Installation and Docs

Install using `go get github.com/hashicorp/errwrap`.

Full documentation is available at
http://godoc.org/github.com/hashicorp/errwrap

## Usage

#### Basic Usage

Below is a very basic example of its usage:

```go
package main

import (
	"fmt"
	"os"

	"github.com/hashicorp/errwrap"
)

// A custom error message
const ErrNotExist = "File not found"

// A function that always returns an error, but wraps it, like a real
// function might.
func tryOpen() error {
	_, err := os.Open("/i/dont/exist")
	if err != nil {
		return errwrap.Wrapf("Doesn't exist: {{err}}", err)
	}

	return nil
}

// A function that wraps two errors
func tryOpen2() error {
	_, err := os.Open("/you/dont/exist")
	if err != nil {
		return errwrap.Wrapf(ErrNotExist, errwrap.Wrapf("Doesn't exist: {{err}}", err))
	}

	return nil
}

func main() {
	fmt.Println("Example 1")
	err := tryOpen()

	// We can use the Contains helpers to check if an error contains
	// another error. It is safe to do this with a nil error, or with
	// an error that doesn't even use the errwrap package.
	if errwrap.Contains(err, ErrNotExist) {
		fmt.Println("  contains os.ErrNotExist")
	}
	if errwrap.ContainsType(err, new(os.PathError)) {
		fmt.Println("  contains type os.PathError")
	}

	// Or we can use the associated `Get` functions to just extract
	// a specific error. This would return nil if that specific error doesn't
	// exist.
	perr := errwrap.GetType(err, new(os.PathError))
	fmt.Println("  " + perr.Error())

	fmt.Println("Example 2")
	err = tryOpen2()

	// Try with two wrapped errors
	if errwrap.Contains(err, ErrNotExist) {
		fmt.Println("  contains os.ErrNotExist")
	}
	if errwrap.ContainsType(err, new(os.PathError)) {
		fmt.Println("  contains type os.PathError")
	}

	perr = errwrap.GetType(err, new(os.PathError))
	fmt.Println("  " + perr.Error())
}
```

#### Custom Types

If you're already making custom types that properly wrap errors, then
you can get all the functionality of `errwraps.Contains` and such by
implementing the `Wrapper` interface with just one function. Example:

```go
type AppError {
  Code ErrorCode
  Err  error
}

func (e *AppError) WrappedErrors() []error {
  return []error{e.Err}
}
```

Now this works:

```go
err := &AppError{Err: fmt.Errorf("an error")}
if errwrap.ContainsType(err, fmt.Errorf("")) {
	// This will work!
}
```
