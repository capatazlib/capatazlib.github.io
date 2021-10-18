# State Management Example

> Note: This example is a very contrived one. The goal is to show the
> capabilities of the library with easy to understand code, not one that would
> make sense.

In our previous [example](./example_1.md), we implemented a `producer` ⟷
`consumer` program that crashed when we mentioned broccoli.

The previous implementation is good enough given the channel used for
communication between our goroutines (shared resource) is never closed. But what
would happen if because of some complicated business logic, our channel gets
closed (invalid state)?

Let us close the channel when we receive the word "cucumber"

```go
package main

import (
    "bufio"
    "context"
    "fmt"
    "os"

    "github.com/capatazlib/go-capataz/cap"
)

func main() {
    buff := make(chan interface{})

    producer := cap.NewWorker("producer", func(ctx context.Context) error {
        reader := bufio.NewReader(os.Stdin)
        for {
            select {
            case <-ctx.Done():
                return nil
            default:
                text, err := reader.ReadString('\n')
                if text == "broccoli\n" {
                    panic("I do not like broccoli")
                }
                // Execute "feature" that closes the communication channel
                if text == "cucumber\n" {
                    close(buff)
                }
                if err != nil {
                    panic(err)
                }
                select {
                case <-ctx.Done():
                    return nil
                case buff <- text:
                }
            }
        }
    })

    consumer := cap.NewWorker("consumer", func(ctx context.Context) error {
        for {
            select {
            case <-ctx.Done():
                return nil
            case msg := <-buff:
                fmt.Printf("received msg: %s\n", msg)
            }
        }
    })

    logEvent := func(ev cap.Event) {
      fmt.Fprintf(os.Stderr, "%v\n", ev)
    }

    spec := cap.NewSupervisorSpec(
      "root",
      cap.WithNodes(producer, consumer),
      cap.WithNotifier(logEvent),
   )

    appCtx := context.Background()
    supervisor, startErr := spec.Start(appCtx)
    if startErr != nil {
        panic(startErr)
    }

    terminationErr := supervisor.Wait()
    if terminationErr != nil {
        fmt.Printf("terminated with errors: %v", terminationErr)
    }
}
```

What do you think is going to happen? Try to guess before looking at the output
bellow.

When we run this program, we get a very interesting output:

```
$ ./example_2
Event{created:  2021-10-18 14:01:58.377600448 -0700 PDT m=+0.000352428, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer}
Event{created:  2021-10-18 14:01:58.378461922 -0700 PDT m=+0.001213907, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/consumer}
Event{created:  2021-10-18 14:01:58.378506302 -0700 PDT m=+0.001258282, tag:       ProcessStarted, nodeTag: Supervisor, processRuntime: root}
hello
received msg: hello

world
received msg: world

broccoli
Event{created:  2021-10-18 14:02:07.077565077 -0700 PDT m=+8.700317164, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/producer, err: panic error: I do not like broccoli}
Event{created:  2021-10-18 14:02:07.078041552 -0700 PDT m=+8.700793536, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer}
cucumber
received msg: %!s(<nil>)
received msg: %!s(<nil>)
... Message repeated infinitely
<Ctrl-C>
```

Oh no, we implemented an infinite loop that reads a closed channel. Maybe if we
check that the channel is closed and terminate the worker, things are going to
work:

```go
package main

import (
  "errors"
  // ...
)

func main() {
    // ...

    consumer := cap.NewWorker("consumer", func(ctx context.Context) error {
        for {
            select {
            case <-ctx.Done():
                return nil
            case msg, ok := <-buff:
                if !ok {
                    return errors.New("consumer chan is closed")
                }
                fmt.Printf("received msg: %s\n", msg)
            }
        }
    })

    // ...
}
```

If we run this program again, is the application going to recover? Let's see:

```
$ ./example_2
Event{created:   2021-10-18 14:05:34.29107699 -0700 PDT m=+0.000542504, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer}
Event{created:  2021-10-18 14:05:34.292069745 -0700 PDT m=+0.001535241, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/consumer}
Event{created:  2021-10-18 14:05:34.292111319 -0700 PDT m=+0.001576815, tag:       ProcessStarted, nodeTag: Supervisor, processRuntime: root}
hello
received msg: hello

world
received msg: world

cucumber
Event{created:   2021-10-18 14:05:38.80534924 -0700 PDT m=+4.514814712, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/consumer, err: Consumer is not there}
Event{created:  2021-10-18 14:05:38.805445809 -0700 PDT m=+4.514911246, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/consumer}
Event{created:  2021-10-18 14:05:38.805459842 -0700 PDT m=+4.514925271, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/producer, err: send on closed channel}
Event{created:  2021-10-18 14:05:38.805490588 -0700 PDT m=+4.514956018, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/consumer, err: Consumer is not there}
Event{created:  2021-10-18 14:05:38.805519651 -0700 PDT m=+4.514985081, tag:        ProcessFailed, nodeTag: Supervisor, processRuntime: root, err: supervisor crashed due to restart tolerance surpassed}
terminated with errors: supervisor crashed due to restart tolerance surpassed
```

Nope, still bad, although better because now we are failing fast. What's
happening here? The application gets in a state that it cannot recover from
because the channel was created _outside_ the supervision tree. Whenever the
supervisor restarts, it is keeping the old state around, which defeats the
purpose of Supervision Trees.

For this use cases where we have a _shared_ state among many workers of a
Supervisor, we use the [`cap.BuildNodesFn`]() function to build all the shared
resources and the supervised workers that are going to use them.

```go
package main

import (
    "bufio"
    "context"
    "errors"
    "fmt"
    "os"

    "github.com/capatazlib/go-capataz/cap"
)

func main() {
    logEvent := func(ev cap.Event) {
        fmt.Fprintf(os.Stderr, "%v\n", ev)
    }

    spec := cap.NewSupervisorSpec(
        "root",
        // Provide a custom cap.BuildNodesFn that:
        //
        // * Setups all the resources and workers in a (re)start callback.
        //
        // * When allocating resources, return a `cap.CleanupFn` that
        //   closes them when the supervisor is terminated.
        //
        // * If there is an error allocating a resource, return the error
        //   and _fail fast_.
        //
        func() (workers []cap.Node, cleanupFn func() error, resourceAcquErr error) {
            // Create buff, producer and consumer inside the cap.BuildNodesFn
            buff := make(chan interface{})

            producer := cap.NewWorker("producer", func(ctx context.Context) error {
              // previous implementation
            })

            consumer := cap.NewWorker("consumer", func(ctx context.Context) error {
              // previous implementation
            })

            // There is no resource allocation, so no need to
            // perform a cleanup
            cleanupFn = func() (resourceCleanupErr error) {
                return nil
            }

            return []cap.Node{producer, consumer}, cleanupFn, nil

        },
        cap.WithNotifier(logEvent),
    )

    appCtx := context.Background()
    supervisor, startErr := spec.Start(appCtx)
    if startErr != nil {
        panic(startErr)
    }

    terminationErr := supervisor.Wait()
    if terminationErr != nil {
        fmt.Printf("terminated with errors: %v", terminationErr)
    }
}
```

What do you think, does it work now?

```
$ ./example_2
Event{created:  2021-10-18 14:29:42.772181434 -0700 PDT m=+0.000405612, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer}
Event{created:  2021-10-18 14:29:42.772920751 -0700 PDT m=+0.001144916, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/consumer}
Event{created:  2021-10-18 14:29:42.772961871 -0700 PDT m=+0.001186054, tag:       ProcessStarted, nodeTag: Supervisor, processRuntime: root}
hello
received msg: hello

cucumber
Event{created:  2021-10-18 14:29:45.167407521 -0700 PDT m=+2.395631796, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/consumer, err: consumer chan is closed}
Event{created:  2021-10-18 14:29:45.167749604 -0700 PDT m=+2.395973777, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/consumer}
Event{created:  2021-10-18 14:29:45.167792201 -0700 PDT m=+2.396016366, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/producer, err: send on closed channel}
Event{created:  2021-10-18 14:29:45.167871787 -0700 PDT m=+2.396095961, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/consumer, err: consumer chan is closed}
Event{created:  2021-10-18 14:29:45.167935431 -0700 PDT m=+2.396159596, tag:        ProcessFailed, nodeTag: Supervisor, processRuntime: root, err: supervisor crashed due to restart tolerance surpassed}
terminated with errors: supervisor crashed due to restart tolerance surpassed
```

Not quite.

How come our `cap.BuildNodesFn` doesn't get called?

When a Supervisor detects to many restarts from its children, it terminates all
the running children, cleans its allocated resources and terminates with an
error. If no other supervisor is there to catch it, the application terminates
with that error.

So, who supervises the supervisor? Another supervisor at the top of course.

```go
package main

import (
    "bufio"
    "context"
    "errors"
    "fmt"
    "os"

    "github.com/capatazlib/go-capataz/cap"
)

func main() {
    logEvent := func(ev cap.Event) {
        fmt.Fprintf(os.Stderr, "%v\n", ev)
    }

    // Make this a subSystem variable that gets used as a child node of a bigger
    // supervision tree.
    subSystem := cap.NewSupervisorSpec(
        "producer-consumer",
        func() (workers []cap.Node, cleanupFn func() error, resourceAcquErr error) {
            buff := make(chan interface{})

            producer := cap.NewWorker("producer", func(ctx context.Context) error {
              // ...
            })

            consumer := cap.NewWorker("consumer", func(ctx context.Context) error {
              // ...
            })

            cleanupFn = func() (resourceCleanupErr error) {
                return nil
            }

            return []cap.Node{producer, consumer}, cleanupFn, nil

        },
    )

    spec := cap.NewSupervisorSpec(
        "root",
        cap.WithNodes(
            // Use `cap.Subtree` to insert this subSystem in our
            // application
            cap.Subtree(subSystem),
        ),
        // Keep the notifier at the top of the supervision tree
        cap.WithNotifier(logEvent),
    )

    appCtx := context.Background()
    supervisor, startErr := spec.Start(appCtx)
    if startErr != nil {
        panic(startErr)
    }

    terminationErr := supervisor.Wait()
    if terminationErr != nil {
        fmt.Printf("terminated with errors: %v", terminationErr)
    }
}


```

Now, let's see if we finally could make this very contrived application reliable:

```
$ ./example_2
Event{created:  2021-10-18 14:40:02.952526855 -0700 PDT m=+0.000340334, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer-consumer/producer}
Event{created:  2021-10-18 14:40:02.952935936 -0700 PDT m=+0.000749413, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer-consumer/consumer}
Event{created:  2021-10-18 14:40:02.952954676 -0700 PDT m=+0.000768141, tag:       ProcessStarted, nodeTag: Supervisor, processRuntime: root/producer-consumer}
Event{created:  2021-10-18 14:40:02.952967745 -0700 PDT m=+0.000781216, tag:       ProcessStarted, nodeTag: Supervisor, processRuntime: root}
hello
received msg: hello

world
received msg: world

cucumber // Crash
Event{created:  2021-10-18 14:40:10.169566747 -0700 PDT m=+7.217380249, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/producer-consumer/consumer, err: consumer chan is closed}
Event{created:  2021-10-18 14:40:10.169711812 -0700 PDT m=+7.217525283, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer-consumer/consumer}
Event{created:  2021-10-18 14:40:10.169732943 -0700 PDT m=+7.217546404, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/producer-consumer/producer, err: send on closed channel}
Event{created:  2021-10-18 14:40:10.169781719 -0700 PDT m=+7.217595185, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/producer-consumer/consumer, err: consumer chan is closed}
Event{created:  2021-10-18 14:40:10.169816451 -0700 PDT m=+7.217629933, tag:        ProcessFailed, nodeTag: Supervisor, processRuntime: root/producer-consumer, err: supervisor crashed due to restart tolerance surpassed}
Event{created:  2021-10-18 14:40:10.169952496 -0700 PDT m=+7.217765976, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer-consumer/producer}
Event{created:  2021-10-18 14:40:10.170026745 -0700 PDT m=+7.217840202, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer-consumer/consumer}
Event{created:  2021-10-18 14:40:10.170038109 -0700 PDT m=+7.217851568, tag:       ProcessStarted, nodeTag: Supervisor, processRuntime: root/producer-consumer}
hello // Back in business
received msg: hello
```

Fresh like a cucumber.

## What did we learn

* Having a supervision tree without resetting state is not a very useful
  supervision tree.

* Use the `cap.BuildNodesFn` function to create tree nodes that use a shared
  resource.

* How the `cap.BuildNodesFn` allows us to return errors on start or termination.

* If we want to ensure a supervisor initialization gets called again, we need to
  supervise our supervisor from another supervisor :yawdawg:.
