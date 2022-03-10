# Go Closer

Go Closer provides utility functions for closing resources in Go.

## Ensuring a resource is always closed

To ensure a resource is always closed in Go, it is common practice to defer a
call to close the resource immediately after opening it successfully. Consider
an `openResource` function with three return values: a resource, a callback
function to close the resource, and an error if the resource could not be
opened. A common practice is to write the following:

```go
func f() error {
  res, closeFn, err := openResource()
  if err != nil {
    return nil, err
  }
  defer closeFn()
  return useResource(res)
}

```

However, this solution is less satisfying in the case that the `closeFn`
callback can itself return an error. In such cases, it is appropriate to use
`closer.Close` or `closer.CloseAndLog` to ensure the resource is closed but the
error is still surfaced in some way. An example use of `closer.Close`:

```go
func f() (rerr error) {
  res, closeFn, err := openResource()
  if err != nil {
    return nil, err
  }
  defer closer.Close(&rerr, closeFn, "error closing resource")
  return useResource(res)
}
```

The deferred `closer.Close` ensures:

1.  That `closeFn` is called regardless of the outcome of `useResource` --
    whether it returns normally, with an error, or raises a panic.
1.  If both `useResource` and `closeFn` return errors, `f` returns the same
    error as `useResource` and the error returned by `closeFn` is logged.
1.  If `closeFn` returns an error but `useResource` does not, `f` returns the
    same error as `closeFn`.

There are cases in which that last property may not be desirable. That is, you
may *not* want to return the error produced by `closeFn` even when `useResource`
has returned normally. Perhaps you agree
with the argument
that returning an error on close should only be done when writing to a resource,
not when reading. Or perhaps the outer function `f` complies with an interface
that doesn't permit returning an error. In these cases, use `closer.CloseAndLog`
to unconditionally log any error returned by `closeFn`:

```go
func f() error {
  res, closeFn, err := openResource()
  if err != nil {
    return nil, err
  }
  defer closer.CloseAndLog(closeFn, "error closing resource")
  return useResource(res)
}
```

## Ensuring a resource is closed only when there's a subsequent error

Sometimes a function intends to leave a resource *open* when it returns normally
but wishes to close it in the case it returns an error. For such cases, use
`closer.CloseOnErr` or `closer.CloseVoidOnErr`. An example:

```go
func f() (val Value, rerr error) {
  res, closeFn, err := openResource()
  if err != nil {
    return nil, err
  }
  defer closer.CloseOnErr(&rerr, closeFn, "error closing resource")
  return wrapResource(res)
}
```
