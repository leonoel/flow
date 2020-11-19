# Flow

A clojure-based standard for representation of logical processes producing multiple values over time.


## Goals

* *composable*. State management required by the protocol must be encapsulated in an immutable value allowing functional composition.
* *performant*. The protocol must have low overhead and not require post-bootstrap allocations in order to limit GC pressure.
* *portable*. The browser is a first-class target, the protocol must not rely on threads or other platform-specific features.
* *versatile*. The protocol must be suitable for both discrete event streaming and continuous propagation of change.


## Prior art

Flow shares several goals with various existing streaming protocols, especially [reactive streams](https://www.reactive-streams.org/), and aims to overcome its limitations.
* *null handling*. Reactive streams is highly opinionated against null, which makes protocol violations common in languages embracing null like clojure. Flow is agnostic about `nil`, so applicative code doesn't have to guard against it.
* *lazy sampling*. Reactive streams forces the producer to eagerly sample continuous signals, i.e transfer current state each time it changes with a keep-latest backpressure strategy. Flow's transfer mechanism makes lazy sampling possible without compromising on backpressure.
* *graceful shutdown*. Reactive streams explicitly forbids post-cancellation communication, thus a producer requiring a non-trivial cleanup procedure has no way to inform the consumer when its resources have been actually released. Flow doesn't make any assumptions about cancelling behavior, which allows for arbitrarily complex graceful shutdown logic.


## Specification

Flow defines a protocol to perform a unidirectional transfer of values from a *producer* to a *consumer*.
* A *flow* is a function provided by the *producer* taking two arguments, a *notifier* and a *terminator*. It is called by the *consumer* to spawn a new instance of the process. It must not block, it must not throw, it must return an *iterator*.
* A *notifier* is a zero-argument function provided by the *consumer*. It is called by the *producer* each time it becomes ready to transfer a value, at which point it must stop signaling until the transfer is confirmed by the *consumer*. It must not be called after the *terminator* has been called. It must not block, it must not throw, its return value should be ignored.
* A *terminator* is a zero-argument function provided by the *consumer*. It is called exactly once by the *producer* when the process instance has no more values to transfer and all of its resources have been released. It must not block, it must not throw, its return value should be ignored.
* An *iterator* is an object provided by the *producer*, it must be callable as a zero-argument function and `deref`able. `deref` is called by the *consumer* to trigger the transfer of a value, at which point the *producer* becomes allowed to signal again. The *consumer* must not call `deref` before the *producer* calls the *notifier*. `deref` must not block and must return the transferred value. It may throw an exception to indicate a failure, in this case it must not call the *notifier* again. The *consumer* may call the *iterator* as a zero-argument function to cancel the process instance. The *producer* must expect this operation to happen at any time including after termination, it must be idempotent, it must not block, it must not throw, its return value should be ignored.
