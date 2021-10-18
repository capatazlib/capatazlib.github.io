# Drop-in Replacement Example

> A few notes:
>
> * This example is not managing state properly for the sake of simplicity/learning.
>
> * Using a Supervision Tree on small applications could be overkill; ensure
>   that the inherited complexity of Supervision Trees is not higher than the
>   complexity of your program.

## Using goroutines

We could use Capataz as a drop-in replacement for regular `go` statements.

Imagine we have a simple application that spawns two goroutines that form a
`producer` ⟷ `consumer` relationship. We are going to add an arbitrary input
error to showcase the restart capabilities of Capataz; every time the user
writes "broccoli" the system is going to crash.

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "sync"
)

func main() {
  buff := make(chan interface{})
  var wg sync.WaitGroup

  // Routine that gets a value from somewhere
  wg.Add(1)
  go func() {
    defer wg.Done()
    reader := bufio.NewReader(os.Stdin)
    for {
      text, err := reader.ReadString('\n')
      if text == "broccoli\n" {
        panic("I do not like broccoli")
      }

      if err != nil {
        panic(err)
      }

      buff <- text
    }
  }()

  // Routine that consumes values for something else
  wg.Add(1)
  go func() {
    defer wg.Done()
    for msg := range buff {
      fmt.Printf("received msg: %s\n", msg)
    }
  }()

  wg.Wait()
}
```

When running this software, the program crashes as soon as you say broccoli:

```
$ ./example_1
hello
received msg: hello

world
received msg: world

broccoli
panic: I do not like broccoli

goroutine 6 [running]:
main.main.func1(0xc000026220, 0xc000024360)
        /home/capataz/example/main.go:22 +0x26d
created by main.main
        /home/capataz/example/main.go:16 +0x9c

```

## Using Capataz

We can use a supervision tree value to get the same behavior, but on a
supervised way:

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

    // Replace a go statement with cap.NewWorker call. Note cap.NewWorker can
    // receive multiple configuration options, so make sure to check it's
    // godoc documentation
    producer := cap.NewWorker("producer", func(ctx context.Context) error {
        // NOTE goroutines in capataz must always deal with a context
        // to account for termination signals, and can return an error to note
        // that they failed signaling a message to it's supervisor
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

    // Replace go statement with cap.NewWorker call
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

    // Replace `sync.WaitGroup` with `cap.NewSupervisorSpec`, like
    // `cap.NewWorker`, this function may receive multiple configuration
    // options, check its godoc documentation for more details.
    spec := cap.NewSupervisorSpec("root", cap.WithNodes(producer, consumer))

    // The code above has not spawned any goroutines yet, but it wired up
    // all the nodes the application needs to function in a static way

    // The Start method triggers the spawning of all the nodes in the
    // supervision tree in pre-order, first producer, then consumer, and
    // finally root (our supervisor).
    appCtx := context.Background()
    supervisor, startErr := spec.Start(appCtx)
    if startErr != nil {
        panic(startErr)
    }

    // Join the current goroutine with the supervisor goroutine, in the
    // situation there is a termination error, it will be notified here.
    terminationErr := supervisor.Wait()
    if terminationErr != nil {
        fmt.Printf("terminated with errors: %v", terminationErr)
    }
}
```

In this application, we have an improved management of the lifecycle of our
goroutines via the enforcement of a `context.Context` value, we also survive the
dreadful broccoli:

```
$ ./example_1
hello
received msg: hello

world
received msg: world

broccoli // <--- no crash
hello
received msg: hello
```

## Errors shouldn't go to the void

If you are like us, you probably hate errors getting completely ignored. We are
able to capture errors that happen in our supervision tree using the
EventNotifier API. Let us change the code above slightly by adding an extra
argument to the `NewSupervisorSpec` call.

```go
func main() {
    // ...
    logEvent := func(ev cap.Event) {
      fmt.Fprintf(os.Stderr, "%v\n", ev)
    }

    spec := cap.NewSupervisorSpec(
      "root",
      cap.WithNodes(producer, consumer),
      cap.WithNotifier(logEvent),
    )
    // ...
}
```

If we run our application again, we get a better idea of what is going on under
the covers:

```
$ ./example_1
Event{created:  2021-10-15 17:14:29.116512659 -0700 PDT m=+0.000185322, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer}
Event{created:  2021-10-15 17:14:29.116846811 -0700 PDT m=+0.000519467, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/consumer}
Event{created:  2021-10-15 17:14:29.116860487 -0700 PDT m=+0.000533142, tag:       ProcessStarted, nodeTag: Supervisor, processRuntime: root}
hello
received msg: hello

world
received msg: world

broccoli // <-- crash
Event{created: 2021-10-15 17:14:42.265600375 -0700 PDT m=+13.149273244, tag:        ProcessFailed, nodeTag:     Worker, processRuntime: root/producer, err: panic error: I do not like broccoli}
Event{created: 2021-10-15 17:14:42.265932578 -0700 PDT m=+13.149605300, tag:       ProcessStarted, nodeTag:     Worker, processRuntime: root/producer}
hello again // <-- back to business
received msg: hello again
```

## What did we learn

* We may use a `NewSupervisorSpec` + `NewWorker` in favor of `go` routines if we
  want to restart them on failure.

* Use the `cap.WithNotifier` to get full visibility on what is going on
  inside the supervision tree.

* Use the `context.Context` API to manage the lifecycle of worker nodes.

[Next](./example_2.md), we are going to initialize and manage our state inside
the root Supervisor runtime.
