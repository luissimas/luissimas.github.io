---
title: 'Graceful shutdown in Go'
date: '2025-10-03'
comments: true
tags:
  - go
---

Recently, I've had to implement a graceful shutdown process in a side project. While doing so, I was pleasantly surprised with how nicely we can implement this using Go's context package. It just felt right.

I think this is a nice example of how Go can make doing the right thing simple. While there is no guarantee that all components within a system will respect context deadlines and handle graceful shutdown, they are well equipped to do so in a reliable and standardized way.

## What is graceful shutdown

Graceful shutdown is the process of terminating an application in a controlled and intentional way. This usually means going through the following steps:

1. Detect the shutdown signal
2. Stop accepting new work
3. Finish the work that is already in progress
4. Perform some cleanup work if required
5. Shutdown

Graceful shutdown applies to any type of application. In this post we'll focus on server-side apps, but the same principles apply to client-side apps like GUIs or CLIs.

## Why should we care

The main reason we care about graceful shutdowns is user experience. When a server-side application is terminated without a graceful shutdown, all active requests are terminated abruptly, resulting in errors to the client.

There's also the issue of releasing resources. The underlying OS is capable of cleaning up the resources such as open sockets and file handles once the process exits, so we generally don't need to worry about those. What we might need to worry about is downstream connections to other services such as databases and message brokers. Many of those systems support gracefully closing open connections, ensuring proper resource cleanup without having to rely on timeouts.

### Modern applications are disposable

It's also worth mentioning the [Twelve Factor Apps's disposability factor](https://12factor.net/disposability). These days, most web server-side applications are usually being managed by some orchestrators such as Kubernetes or a PaaS platform. These orchestrators can start, stop and restart instances of our applications for several reasons such as autoscaling rules, new deployments, etc.

This means that we should expect the possibility of our applications to be started and stopped several times a day. If every time our application needs to shutdown it causes client-side errors, our users will experience those errors several times a day.

## Implementation

To implement graceful shutdown in Go applications, we can rely on the mechanisms provided by the [context package](https://pkg.go.dev/context). A graceful shutdown process is fundamentally about cancellation, so the context package is a great fit here.

### Catching the signal

The first thing we need to do is detect that our application needs to shutdown. For decades, signals have been the standard way to detect that in UNIX systems. This standard is usually followed by orchestrators. For example, Kubernetes will send a `SIGTERM` when starting the shutdown process for a pod.

The signals we want to listen for are `SIGTERM` and `SIGINT`. `SIGTERM` is the standard termination signal, this is what will be sent to the process when a shutdown should be performed. `SIGINT` is the signal sent when a keyboard interrupt occurs (i.e. we press Ctrl+C in the shell that is running our process). Handling `SIGTERM` is essential here. Handling `SIGINT` is not required, but useful for local development.

In Go, we can use the `signal.NotifyContext` function to create a new context that will be canceled once the process receives any of the specified signals.

```go
ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
defer cancel()
```

### Starting the shutdown

We can use the context provided by `signal.NotifyContext` to determine when to start our shutdown process. As a simple example: let's say we have a program that does some work every second, waiting on a `time.Ticker`. We can then wait on both the ticker and the context channels to either perform the next iteration of work or start the shutdown process

```go
func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
	defer cancel()
	
	ticker := time.NewTicker(1 * time.Second)
	for {
		select {
		case <-ctx.Done():
			slog.Info("Stopping work.")
			// Do some cleanup here
			return
		case <-ticker.C:
			slog.Info("Doing some work")
			// Do some actual work here
		}
	}
}
```

With this, we have everything we need to implement a basic graceful shutdown process in any Go application.

This approach with `time.Ticker` is perfectly suitable for a simple interval-based worker process. In fact, this is exactly the approach I used in my [zettelkasten-exporter](https://github.com/luissimas/zettelkasten-exporter) project. Still, I think it's also worth showing an example of a more common type of application.

### An HTTP server example

To handle graceful shutdowns in an HTTP server, we'll be leveraging the `Shutdown` function of the `http.Server` in the [net/http](https://pkg.go.dev/net/http) package. We'll still have to catch the signal to determine when to start the shutdown process, but the `Shutdown` function will handle the process of stopping to accept new requests and waiting for active requests to finish.

To start this example, let's define a few constants. Our server will listen on `localhost:8080`, and we'll define two durations: one for the duration of the http handler processing (to simulate slow requests), and other for the maximum time we want to wait for active requests to finish.

```go {filename="main.go"}
const (
	serverAddr          = "localhost:8080"
	httpHandlerDuration = 10 * time.Second
	httpShutdownTimeout = 10 * time.Second
)
```

We'll create our server in a `newServer` function. It simply exposes a `GET /` handler that sleeps for the duration of `httpHandlerDuration` and then responds with some data. The server has some basic timeout configuration which is not really relevant for this example.

```go {filename="main.go"}
func newServer(addr string) http.Server {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(httpHandlerDuration)
		w.Write([]byte("Some data"))
	})

	return http.Server{
		Addr:         addr,
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  120 * time.Second,
	}
}
```

In our `main` function, the first thing we do is setup a context tied to both `SIGTERM` and `SIGINT` signals.

```go {filename="main.go"}
func main() {
	signalCtx, cancel := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
	defer cancel()
	// ...
}
```

Then, we create our new server and start the listener loop. Since `ListenAndServe` blocks the caller, we want to call it in a new goroutine to leave the main goroutine free to wait on the `signalCtx`.

```go {filename="main.go"}
func main() {
	// ...
	server := newServer(serverAddr)
	go func() {
		slog.Info("Server listening for requests", slog.String("addr", serverAddr))
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			slog.Error("Error on server listener", slog.Any("error", err))
		}
	}()
	// ...

```

After starting the server, the main goroutine simply waits on the channel returned by `signalCtx.Done()`. This means that it will block until the process receives a `SIGTERM` or `SIGINT` signal.

After receiving the signal, we start the shutdown process by calling `Shutdown` on the server. Notice that we also create a new `shutdownCtx` with `context.WithTimeout` and pass it to `Shutdown`. This provides a timeout to wait for active requests to finish.

With this, the `Shutdown` call will:

1. Stop the server from accepting new requests by closing the underlying listener socket
2. Wait for all active requests to finish or for `shutdownCtx` to expire, whichever happens first
3. Return any error in the process, either from closing the listeners or from the `shutdownCtx`

```go {filename="main.go"}
func main() {
	// ...
	<-signalCtx.Done()

	slog.Info("Starting shutdown process")
	shutdownCtx, cancel := context.WithTimeout(context.Background(), httpShutdownTimeout)
	defer cancel()
	if err := server.Shutdown(shutdownCtx); err != nil {
		slog.Error("Error on server shutdown", slog.Any("error", err))
	}

	slog.Info("Shutdown complete")
}
```

But even here we have a problem: what happens if `shutdownCtx` expires but there are still active requests being processed? As it is now, the process will simply terminate, and those request will receive an error.

To handle these cases, we have a trade-off to make between ensuring that all requests get processed and ensuring that our process finishes within a known time limit. The two main options I see are:

1. Use a base context for all incoming requests (setting the `BaseContext` field in `http.Server` and then retrieving it in the handler with `r.Context()`), cancel it once `server.Shutdown` returns with a `context.DeadlineExceeded` error and wait a few extra seconds before exiting. This gives the handlers a chance to cancel downstream calls and perform any cleanup before returning a suitable timeout error to the client.
2. Use a `sync.WaitGroup` for all incoming requests, and make the main goroutine wait on this group to ensure that all requests get completely processed before finally shutting down. In this case, we might need to assume that the process can wait for an indefinite amount of time before shutting down.

Usually we want to ensure that our process can be shutdown in a reasonable amount of time, so I'd go with option 1 for most use cases. But this begs the question: how long should we wait for active requests to finish? This ultimately depends on your environment, how long your requests usually take to complete, how quickly you need to shutdown, etc. In Kubernetes for example, we can set the `terminationGracePeriodSeconds` in the pod spec to control how much time the process has to shutdown gracefully after receiving a `SIGTERM`. By default this value is 30 seconds.

## Putting it all together

Here's the final version of our HTTP server example with some comments in case you want to play around with it.

```go {filename="main.go"}
const (
	serverAddr          = "localhost:8080"
	httpHandlerDuration = 10 * time.Second
	httpShutdownTimeout = 10 * time.Second
)

func newServer(addr string) http.Server {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(httpHandlerDuration)
		w.Write([]byte("Some data"))
	})

	return http.Server{
		Addr:         addr,
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  120 * time.Second,
	}
}

func main() {
	// Setup the context that will be tied to external signals
	signalCtx, cancel := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
	defer cancel()

	// Start the server loop in a different goroutine
	server := newServer(serverAddr)
	go func() {
		slog.Info("Server listening for requests", slog.String("addr", serverAddr))
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			slog.Error("Error on server listener", slog.Any("error", err))
		}
	}()

	// Wait for shutdown signal in the main goroutine
	<-signalCtx.Done()

	// Shutdown server with a timeout
	slog.Info("Starting shutdown process")
	shutdownCtx, cancel := context.WithTimeout(context.Background(), httpShutdownTimeout)
	defer cancel()
	if err := server.Shutdown(shutdownCtx); err != nil {
		slog.Error("Error on server shutdown", slog.Any("error", err))
	}

	// We can do some cleanup here

	slog.Info("Shutdown complete")
}
```

## That's it

I hope I was able to illustrate the basic mechanisms of implementing graceful shutdowns in Go applications. Personally, I really like how the context package can be used for this purpose. We have a single and flexible `context.Context` for cancellation that we can use. But the interface itself says nothing about how this cancellation is triggered. It can be a timeout (`context.WithTimeout`), an explicit cancellation (`context.WithCancel`) or even an OS signal (`signal.NotifyContext`)!

We didn't get into much detail about the cleanup process, as this is highly specific to which downstream services your application uses. But the general idea should apply regardless of those specifics.

Thanks for sticking around!

## Links

Nothing here is new knowledge. I've learned a lot from these sources before writing this post. Check them out if you want to dive deeper into the subject!

- [The Twelve-Factor App: Disposability](https://12factor.net/disposability)
- [Graceful Shutdown in Go (YouTube video)](https://www.youtube.com/watch?v=lak8Xw1GNl0)
- [Go graceful shutdown: best practices and examples (VictoriaMetrics Blog)](https://victoriametrics.com/blog/go-graceful-shutdown/)
- [Kubernetes Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
