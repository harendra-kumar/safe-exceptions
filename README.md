# safe-exceptions

*Safe, consistent, and easy exception handling*

[![Build Status](https://travis-ci.org/fpco/safe-exceptions.svg?branch=master)](https://travis-ci.org/fpco/safe-exceptions)

Runtime exceptions - as exposed in `base` by the `Control.Exception`
module - have long been an intimidating part of the Haskell
ecosystem. This package, and this README for the package, are intended
to overcome this. By providing an API that encourages best practices,
and explaining the corner cases clearly, the hope is to turn what was
previously something scary into an aspect of Haskell everyone feels
safe using.

## Quickstart

This section is intended to give you the bare minimum information to
use this library (and Haskell runtime exceptions in general)
correctly.

* Import the `Control.Exception.Safe` module. Do _not_ import
  `Control.Exception` itself, which lacks the safety guarantees that
  this library adds. Same applies to `Control.Monad.Catch`.
* If something can go wrong in your function, you can report this with
  the `throw`. (For compatible naming, there are synonyms for this of
  `throwIO` and `throwM`.)
* If you want to catch a specific type of exception, use `catch`,
  `handle`, or `try`.
* If you want to recover from _anything_ that may go wrong in a
  function, use `catchAny`, `handleAny`, or `tryAny`.
* If you want to launch separate threads and kill them externally, you
  should use the
  [async package](https://www.stackage.org/package/async).
* Unless you really know what you're doing, avoid the following functions:
    * `catchAsync`
    * `handleAsync`
    * `tryAsync`
    * `impureThrow`
    * `throwTo`
* If you need to perform some allocation or cleanup of resources, use
  one of the following functions (and _don't_ use the
  `catch`/`handle`/`try` family of functions):
    * `onException`
    * `withException`
    * `bracket`
    * `bracket_`
    * `finally`
    * `bracketOnError`
    * `bracketOnError_`

Hopefully this will be able to get you up-and-running quickly.

_Request to readers_: if there are specific workflows that you're
unsure of how to accomplish with this library, please ask so we can
develop a more full-fledged cookbook as a companion to this file.

## Terminology

We're going to define three different versions of exceptions. Note
that these definitions are based on _how the exception is thrown_, not
based on _what the exception itself is_:

* **Synchronous** exceptions are generated by the current
  thread. What's important about these is that we generally want to be
  able to recover from them. For example, if you try to read from a
  file, and the file doesn't exist, you may wish to use some default
  value instead of having your program exit, or perhaps prompt the
  user for a different file location.

*   **Asynchronous** exceptions are thrown by either a different user
    thread, or by the runtime system itself. For example, in the
    `async` package, `race` will kill the longer-running thread with
    an asynchronous exception. Similarly, the `timeout` function will
    kill an action which has run for too long. And the runtime system
    will kill threads which appear to be deadlocked on `MVar`s or
    `STM` actions.

    In contrast to synchronous exceptions, we almost never want to
    recover from asynchronous exceptions. In fact, this is a common
    mistake in Haskell code, and from what I've seen has been the
    largest source of confusion and concern amongst users when it
    comes to Haskell's runtime exception system.

*   **Impure** exceptions are hidden inside a pure value, and exposed
    by forcing evaluation of that value. Examples are `error`,
    `undefined`, and `impureThrow`. Additionally, incomplete pattern
    matches can generate impure exceptions. Ultimately, when these
    pure values are forced and the exception is exposed, it is thrown
    as a synchronous exception.

    Since they are ultimately thrown as synchronous exceptions, when
    it comes to handling them, we want to treat them in all ways like
    synchronous exceptions. Based on the comments above, that means we
    want to be able to recover from impure exceptions.

## Why catch asynchronous exceptions?

If we never want to be able to recover from asynchronous exceptions,
why do we want to be able to catch them at all? The answer is for
_resource cleanup_. For both sync and async exceptions, we would like
to be able to acquire resources - like file descriptors - and register
a cleanup function which is guaranteed to be run. This is exemplified
by functions like `bracket` and `withFile`.

So to summarize:

* All synchronous exceptions should be recoverable
* All asynchronous exceptions should not be recoverable
* In both cases, cleanup code needs to work reliably

## Determining sync vs async

Unfortunately, GHC's runtime system provides no way to determine if an
exception was thrown synchronously or asynchronously, but this
information is vitally important. There are two general approaches to
dealing with this:

* Run an action in a separate thread, don't give that thread's ID to
  anyone else, and assume that any exception that kills it is a
  synchronous exception. This approach is covered in the School of
  Haskell article
  [catching all exceptions](https://www.schoolofhaskell.com/user/snoyberg/general-haskell/exceptions/catching-all-exceptions).

* Make assumptions based on the type of an exception, assuming that
  certain exception types are only thrown synchronously and certain
  only asynchronously.

Both of these approaches have downsides. For the downsides the
type-based approach, see the caveats section at the end. The problems
with the first are more interesting to us here:

* It's much more expensive to fork a thread every time we want to deal
  with exceptions
* It's not fully reliable: it's possible for the thread ID of the
  forked thread to leak somewhere, or the runtime system to send it an
  async exception
* While this works for actions living in `IO`, it gets trickier for
  pure functions and monad transformer stacks. The latter issue is
  solved via monad-control and the exceptions packages. The former
  issue, however, means that it's impossible to provide a universal
  interface for failure for pure and impure actions. This may seem
  esoteric, and if so, don't worry about it too much.

Therefore, this package takes the approach of trusting type
information to determine if an exception is asynchronous or
synchronous. The details are less interesting to a user, but the
basics are: we leverage the extensible extension system in GHC and
state that any extension type which is a child of `SomeAsyncException`
is an async exception. All other exception types are assumed to be
synchronous.

## Handling of sync vs async exceptions

Once we're able to distinguish between sync and async exceptions, and
we know our goals with sync vs async, how we handle things is pretty
straightforward:

* If the user is trying to install a cleanup function (such as with
  `bracket` or `finally`), we don't care if the exception is sync or
  async: call the cleanup function and then rethrow the exception.
* If the user is trying to catch an exception and recover from it,
  only catch sync exceptions and immediately rethrow async exceptions.

With this explanation, it's useful to consider async exceptions as
"stronger" or more severe than sync exceptions, as the next section
will demonstrate.

## Exceptions in cleanup code

One annoying corner case is: what happens if, when running a cleanup function after an exception was thrown, the cleanup function _itself_ throws an exception. For this, we'll consider ``action `onException` cleanup``. There are four different possibilities:

* `action` threw sync, `cleanup` threw sync
* `action` threw sync, `cleanup` threw async
* `action` threw async, `cleanup` threw sync
* `action` threw async, `cleanup` threw async

Our guiding principle is: we cannot hide a more severe exception with
a less severe exception. For example, if `action` threw a sync
exception, and then `cleanup` threw an async exception, it would be a
mistake to rethrow the sync exception thrown by `action`, since it
would allow the user to recover when that is not desired.

Therefore, this library will always throw an async exception if either
the action or cleanup thows an async exception. Other than that, the
behavior is currently undefined as to which of the two exceptions will
be thrown. The library reserves the right to throw away either of the
two thrown exceptions, or generate a new exception value completely.

## Typeclasses

The [exceptions package](https://www.stackage.org/package/exceptions)
provides an abstraction for throwing, catching, and cleaning up from
exceptions for many different monads. This library leverages those
type classes to generalize our functions.

## Naming

There are a few choices of naming that differ from the base libraries:

* `throw` in this library is for synchronously throwing within a
  monad, as opposed to in base where `throwIO` serves this purpose and
  `throw` is for impure throwing. This library provides `impureThrow`
  for the latter case, and also provides convenience synonyms
  `throwIO` and `throwM` for `throw`.
* The `catch` function in this package will not catch async
  exceptions. Please use `catchAsync` if you really want to catch
  those, though it's usually better to use a function like `bracket`
  or `withException` with ensure that the thrown exception is
  rethrown.

## Caveats

Let's talk about the caveats to keep in mind when using this library.

### Checked vs unchecked

There is a big debate and difference of opinion regarding checked
versus unchecked exceptions. With checked exceptions, a function
states explicitly exactly what kinds of exceptions it can throw. With
unchecked exceptions, it simply says "I can throw some kind of
exception." Java is probably the most famous example of a checked
exception system, with many other languages (including C#, Python, and
Ruby) having unchecked exceptions.

As usual, Haskell makes this interesting. Runtime exceptions are most
assuredly unchecked: all exceptions are converted to `SomeException`
via the `Exception` typeclass, and function signatures do not state
which specific exception types can be thrown (for more on this, see
next caveat). Instead, this information is relegated to documentation,
and unfortunately is often not even covered there.

By contrast, approaches like `ExceptT` and `EitherT` are very explicit
in the type of exceptions that can be thrown. The cost of this is that
there is extra overhead necessary to work with functions that can
return different types of exceptions, usually by wrapping all possible
exceptions in a sum type.

This library isn't meant to settle the debate on checked vs unchecked,
by rather to bring sanity to Haskell's runtime exception system. As
such, this library is decidedly in the unchecked exception camp,
purely by virtue of the fact that the underlying mechanism is as well.

### Explicit vs implicit

Another advantage of the `ExceptT`/`EitherT` approach is that you are
explicit in your function signature that a function may fail. However,
the reality of Haskell's standard libraries are that many, if not the
vast majority, of `IO` actions can throw some kind of exception. In
fact, once async exceptions are considered, _every_ `IO` action can
throw an exception.

Once again, this library deals with the status quo of runtime
exceptions being ubiquitous, and gives the rule: you should consider
the `IO` type as meaning _both_ that a function modifies the outside
world, _and_ may throw an exception (and, based on the previous
caveat, may throw _any type_ of exception it feels like).

There are attempts at alternative approaches here, such as
[unexceptionalio](https://www.stackage.org/package/unexceptionalio). Again,
this library isn't making a value statement on one approach versus
another, but rather trying to make today's runtime exceptions in
Haskell better.

### Type-based differentiation

As explained above, this library makes heavy usage of type information
to differentiate between sync and async exceptions. While the approach
used is fairly well respected in the Haskell ecosystem today, it's
certainly not universal, and definitely not enforced by the
`Control.Exception` module. In particular, `throwIO` will allow you to
synchronously throw an exception with an asynchronous type, and
`throwTo` will allow you to asynchronously throw an exception with a
synchronous type.

The functions in this library prevent that from happening via
exception type wrappers, but if an underlying library does something
surprising, the functions here may not work correctly. Further, even
when using this library, you may be surprised by the fact that `throw
Foo >>= catch (\Foo -> ...)` won't actually trigger the exception
handler if `Foo` looks like an asynchronous exception.

The ideal solution is to make a stronger distinction in the core
libraries themselves between sync and async exceptions.

### Deadlock detection exceptions

Two exceptions types which are handled surprisingly are
`BlockedIndefinitelyOnMVar` and `BlockedIndefinitelyOnSTM`. Even
though these exceptions are thrown asynchronously by the runtime
system, for our purposes we treat them as synchronous. The reasons are
twofold:

* There is a specific action taken in the local thread - blocking on a
  variable which will never change - which causes the exception to be
  raised. This makes their behavior very similar to synchronous
  exceptions. In fact, one could argue that a function like `takeMVar`
  is synchronously throwing `BlockedIndefinitelyOnMVar`
* By our standards of recoverable vs non-recoverable, these exceptions
  certainly fall into the recoverable category. Unlike an intentional
  kill signal from another thread or the user (via Ctrl-C), we would
  like to be able to detect the we entered a deadlock condition and do
  something intelligent in an application.
