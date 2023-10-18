---
layout: post
title: "Terraform in the browser"
categories: blog
tags: ["terraform", "wasm"]
published: true
comments: true
excerpted: |
  How we brings Terraform to run in the browser, with the power of Webassembly
script: [post.js]
---

{: toc}

# Background

In the 2023 Microsoft internal hackathon event, I and two of my team mates are started a project trying to bring Terraform to run in the browser, which is a good opportunity to explore the webasembly area, which I haven't stepped into before.

To make things clear, what we actually were doing is not bring the `terraform` client to run in the browser, but a subset of functionality of it. E.g. in this event we turns the [`terraform-client-import`](https://github.com/magodo/terraform-client-go/tree/main/cmd/terraform-client-import) tool into WASM, which implements a subset of functionality of `terraform`, that is able to import then read a resource, and print its state to the stdout. On the other hand, we want the full feature of the terraform provider to be converted to WASM and run in the browser.

Both the client and the plugin (i.e. the server) projects are built on top of `github.com/hashicorp/go-plugin` project. *It is a Go plugin system over RPC. It is the plugin system that has been in use by HashiCorp tooling for over 4 years. While initially created for Packer, it is additionally in use by Terraform, Nomad, Vault, Boundary, and Waypoint*. Ideally, we want all the changes we made are mostly within this project, and the downstream applications (the `terraform-client-import`, the terraform providers) just need to update their dependencies to the WASM version of the go-plugin, and everyhing should works out of the box. Meanwhile, we want to make the changes coexists with the existing implementations, by using Go's build constraints.

# Cranking `go-plugin`

Now, let's see what `go-plugin` actually functions in both client's & server's view. We'll mainly discuss about the startup process below.

## `go-plugin` (CMD runner)

Firstly, we look at how it works at the CLI environment (instead of the browser). The overall process is as illustrated below:

![cmd-start](/assets/img/tf-in-browser/cmd-start.png) 

### Client

For client, the code structure is as below:

```go
client := plugin.NewClient(&plugin.ClientConfig{
    HandshakeConfig: handshakeConfig,
    Plugins:         pluginMap,
    Cmd:             exec.Command("./plugin/greeter"),
    Logger:          logger,
})

// Connect via RPC
rpcClient, _ := client.Client()
// Request the plugin
raw, _ := rpcClient.Dispense("greeter")

fmt.Println(raw.(shared.Greeter).Greet())
```

The `plugin.ClientConfig` is the user provided configuration of the client, explained below:

- `HandshakeConfig`: A fool-proof mechanism to make sure the client is reaching to the expected server, which means their handshake config should match
- `Plguins`: A map of plugins the client needs to dispense (by name)
- `Cmd`: The `exec.Command` that directs to the plugin binary. This will be later used by the `cmdrunner.CmdRunner`, which in turns will call `cmd.StdoutPipe` and `cmd.StderrPipe` to pipe the plugin's stdout/stderr back to the client (we'll talk about the details later)


The `client.Client()` does the actual work that starts and connects to the plugin. The main flow is illustrated as below:

```go
// Pass some config to plugin via env and start it
cmd.Env = append(cmd.Env, env...)
runner, _ = cmdrunner.NewCmdRunner(c.logger, cmd)
runner.Start()

// Start a goroutine to stream plugin stderr to client c.config.Stderr (if set) and client logger
go func() {
	for {
		line := runner.Stderr().ReadLine()
		c.config.Stderr.Write(line)
        l.Debug(line)
    }
}()


// Start a goroutine to strem plguin stdout to client, which streams plugin handshake to client (including the plugin address)
go func() {
    scanner := bufio.NewScanner(runner.Stdout())
    for scanner.Scan() {
        linesCh <- scanner.Text()
    }
}()

// Parse the handshake
line := <-linesCh:
protocol, address, _ := parseHandshake(line)

// Instantiate a rpc client (netRPC/gRPC)
switch c.protocol {
case ProtocolNetRPC:
    c.client = newRPCClient(c)
case ProtocolGRPC:
    c.client = newGRPCClient(c.doneCtx, c)
}
```

For either the `newRPCClient` and the `newGRPCClient`, they were doing similar things:

- Connect to the plugin (dial its addrss)
- Create the rpc client based on the connection above, multiplexing it via either `github.com/hashicorp/yamux` or `gRPCBroker`
- (gr) Copy from the stdout/err conns above to client `client.config.SyncStdout` nad `client.config.SyncStderr`


### Server


For server, the code structure is as below:

```go
checkMagicKeyVaule()
protoVersion, protoType, pluginSet := protocolVersion(opts)


logger := opts.Logger
if logger == nil {
    // internal logger to os.Stderr
    logger = hclog.New(&hclog.LoggerOptions{
        Level:      hclog.Trace,
        // The os.Stderr here is the original stderr, which streams to parent process by the `cmd.Stderr`.
        Output:     os.Stderr,
        JSONFormat: true,
    })
}

// Create a listener either via tcp or uds
listener := serverListener(...)


// Two pairs of pipes are created: the write end is reassigned as the os.Stdout/err later
stdout_r, stdout_w := os.Pipe()
stderr_r, stderr_w := os.Pipe()


// Instantiate the rpc server
var server ServerProtocol
switch protoType {
case ProtocolNetRPC:
    server = &RPCServer{
        Plugins: pluginSet,
        Stdout:  stdout_r,
        Stderr:  stderr_r,
        //...
    }

case ProtocolGRPC:
    // Create the gRPC server
    server = &GRPCServer{
        Plugins: pluginSet,
        Server:  opts.GRPCServer,
        Stdout:  stdout_r,
        Stderr:  stderr_r,
        //...
    }
}

// Write the handshake msg to stdout, which streams to parent process by the `cmd.Stdout`
fmt.Printf("%d|%d|%s|%s|%s|%s\n",
    CoreProtocolVersion,
    protoVersion,
    listener.Addr().Network(),
    listener.Addr().String(),
    protoType,
    serverCert)

// Reassign os.Stdout and os.Stderr, any write to it will
// be piped to its read end, which is then passed to the rpc server, which in turns will stream to the rpc client side,
// which in turns will be copied to the client config's SyncStdout/err.
os.Stdout = stdout_w
os.Stderr = stderr_w


// (gr) Loop and handle client connections 
go server.Serve(listener)
<-doneCh
```

## `go-plugin` (WASM runner)

The challenge for porting the `go-plugin` to run in the browser is that there are some basic functionalities of current architecture are not available in the browser, namely:

- There is no way for the client to fork a execute a child process (i.e. `exec.Command`)
- Many file system related APIs are not available, including the access to `os.Stdxxx`, `open()`, `pipe()`, etc.
- Some network related APIs, especially those server related ones are not available, e.g. `listen()`, `accept()`, etc.

With that kept in mind, we did some investigation and found that the [Web Worker](https://html.spec.whatwg.org/multipage/workers.html) API fits the `go-plugin` model quite well. So we can change the overview of startup process to something like below:

![wasm-start](/assets/img/tf-in-browser/wasm-start.png)

In terms of the communication model, we tries to keep the application protocol layer untouched, i.e. not changing the implementation of netRPC/gRPC client and server at all. Whilst, we are only switching the transport layer from tcp/uds to the bidirectional channel of web workers. 

Before we look into the details of what we actually have changed, I'd like to mention that the web worker related functionalities are encapuslated in a separate Go module: https://github.com/magodo/go-wasmww. A brief introduction to this module is cited from it's own README:

> At its basic, it abstracts the `exec.Cmd` structure, to allow the main thread (*the parent process*) to create a web worker (*the child process*), by specifying the WASM URL, together with any arguments or environment variables, if any.
The main types for this basic usage are:

- `WasmWebWorker`: Used in the main thread, for creating a Dedicated Web Worker
- `WasmSharedWebWorker`: Used in the main thread, for creating a Shared Web Worker

For the application running inside the worker, users are expected to use the `worker.GlobalSelf` and `sharedworker.GlobalSelf` in the package `github.com/magodo/go-webworkers`.

On top of this basic abstraction, we've added the support for the Web Worker connections. In that it supports initialization sync, controlling the peer (e.g. close the peer), and piping the stdout/stderr from the Web Worker back to the outside.

The main types for the connections are:

- `WasmWebWorkerConn`: Used in the main thread, for creating a *connected* Dedicated Web Worker
- `SelfConn`: Used in the Dedicated Web Worker
- `WasmSharedWebWorkerConn`: Used in the main thread, for creating a *connected* Shared Web Worker
- `SelfSharedConn`: Used in the Shared Web Worker

Now let's see how we integrate the web worker into `go-plugin`.

### Client

Firstly, similar as the `cmdrunner`, we've created a separate package called `wasmrunner`, which implements the Runner interface, but using a wasm file (instead of a OS executable), and starts the target plugin as a Web Shared Worker.

For the client `Start()` method, the main flows are quite the same, except two parts:

- Instead of using the `cmdrunner`, we used the `wasmrunner` here
- Instead of resolving the plugin address as a tcp/uds address, we parse it as a self-defined wasm address, which is of the form `name:url`. Since we are using Shared Workers, as long as the `url` and `name` are kept unchanged, then we can obtain a reference to that same worker and communicate with it. See: https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker.

Additionally, we need to change the `Dialer` functions for both netRPC and gRPC, to use the above address (`name:url`) to create a new reference to the same worker (the plugin), which actually *connect*s to the plugin. These functions return a `net.Conn`, which we also need to implement by ourselves. Following lists its basic read and write methods:

```go
func (conn *WebWorkerConn) Write(b []byte) (n int, err error) {
	arraybuf, err := safejs.MustGetGlobal("Uint8Array").New(len(b))
	if err != nil {
		return 0, nil
	}
	n, err = safejs.CopyBytesToJS(arraybuf, b)
	if err != nil {
		return 0, nil
	}
	if n != len(b) {
		return 0, fmt.Errorf("CopyBytesToJS expect to copy %d bytes, actually %d bytes", len(b), n)
	}
	if err := conn.postFunc(arraybuf, nil); err != nil {
		return 0, err
	}
	return len(b), nil
}

func (conn *WebWorkerConn) Read(b []byte) (n int, err error) {
	// ...
}
```

The `Write()` is straight forwards, where it copy the Go bytes to JS `Uint8Array`, and then using the web worker `postMessage()` API to send the payload to the peer. On the `Read()` end, it reads the message event from the peer web worker from the channel, then converts it to Go bytes, meanwhile, tolerates the case that the received message is larger than the provided buffer.

### Server

For the server side, instead of starting a tcp/uds network listener, we started our self-defined wasm listener. This is intuitive as for the nature of shared web worker, it will always register a event handler for the connet events from the outside:

```go
func NewWebWorkerListener() (net.Listener, error) {
	self, err := wasmww.NewSelfSharedConn()
	if err != nil {
		return nil, err
	}
	ch, err := self.SetupConn()
	if err != nil {
		return nil, err
	}
	return &WebWorkerListener{
		self: self,
		ch:   ch,
	}, nil
}

func (l *WebWorkerListener) Accept() (net.Conn, error) {
	port, ok := <-l.ch
	if !ok {
		return nil, net.ErrClosed
	}

	return NewWebWorkerConnForServer(port)
}
```

Instead of using the `os.Pipe()` to create the two pairs of pipes for stdout and stderr, we instead used `chanio.Pipe()` (from `github.com/magodo/chanio`), which creates a pair of channels to mimic the pipes. The write ends are then *redicted* by reimplementing the `writeSync` (was defined by the `wasm_exec.js` glue code) used by the Go code's `write`, which in turns is used by any write to the `os.Stdout` or `os.Stderr` in WASM context. Instead of the original one that invokes the `console.log()`, it just writes to the write end of the `chanio.Pipe`.

## `go-plugin` summary

This is pretty much what we've done for supporting wasm for the `go-plugin`. Some of the heavy works are done in the `go-wasmww` module, while the changes for the `go-plugin` are kept clean and managable. Those code  that need WASM adoption are moved out to a file appended with `_other.go`, with no change. Meanwhile, there are new files with the same prefix, but ends with `_wasm.go`, that contains the wasm implementation. These files are conditionally built by using Go build constraint.

A little more words about the stdout/stderr streaming flows.

For CLI version, it is illustrated below:

![cmd stdstream](/assets/img/tf-in-browser/cmd-stdstream.png)

The logger instantiated in the plugin will write to the plugin process's original stderr, which is piped to the `cmdrunner.Stderr()`, and fed to the client sides logger and `client.config.Stderr`.

Right after the logger instantiation, the server code will reassign the `os.Stdout`/`os.Stderr` to a pipe's write end, whose read end is streamed to the client's `client.config.SyncStdxxx` via RPC. This covers those explicit writes to stdout/stderr (e.g. via fmt.Fprint()).

For WASM version, it is illustrated below:

![wasm stdstream](/assets/img/tf-in-browser/wasm-stdstream.png)

Since when using WASM, we are not reassigning the `os.Stdout`/`os.Stderr`, but reimplement the `writeSync()` used underlying. This impacts not only the explict write to `os.Stdout`/`os.Stderr`, it also impacts the logger instantiated before (as it ultimately will reach to the `wrtieSync()`). This makes everything routes to the `client.config.SyncStdxxx`. As a result, the client has to define the `SyncStdout` and `SyncStderr` in the `ClientConfig`, so that it can read the stdout/stderr from the plugin, including the logs that are printed via log (log-like) API (so we can't differentiate them).

# Cranking the client and the provider

Both only requires one line change in the `go.mod` to use the WASM supportive `go-plugin`: https://github.com/magodo/go-plugin/tree/wasm (though some huge provider might causes WASM build failure due to its size versus the limited virtual space of WASM).
