---
author: "Sarthak Makhija"
title: "Many flavors of Networking IO"
date: 2024-05-22
description: "
The foundation of any networked application hinges on its ability to efficiently handle data exchange. 
But beneath the surface, there's a hidden world of techniques for managing this communication. 
This article dives into various \"flavors\" of networking IO, exploring the trade-offs associated with each approach. 
We'll delve into techniques like Single-Threaded Blocking IO, Multi-Threaded Blocking IO, Non-blocking with Busy Wait, and Single-Threaded Event loop.
"
tags: ["TCP", "Networking", "Golang", "Event loop"]
thumbnail: "/flavors-of-networking-title.webp"
caption: "Background by Bruno Thethe on Pexels"
---

### Introduction

The foundation of any networked application hinges on its ability to efficiently handle data exchange.
But beneath the surface, there's a hidden world of techniques for managing this communication.
This article dives into various "flavors" of networking IO, exploring the trade-offs associated with each approach.

To illustrate various ways applications handle network traffic, we'll build a TCP server using four distinct approaches: 
**blocking I/O with a single thread**, **blocking I/O with multiple threads**, **non-blocking I/O with busy waiting**, and 
**a single-threaded event loop**. 

Each approach offers unique advantages and drawbacks, and by constructing a server for each approach, we'll gain a deeper 
understanding of their strengths and weaknesses. 

### Overview of our TCP Server

Our TCP server operates on messages encoded using the [protobuf](https://protobuf.dev/) format, specifically the [KeyValueMessage](https://github.com/SarthakMakhija/many-flavors-of-networking-io/blob/main/single_thread_blocking_io/proto/key_value_message.proto). 
These messages can be either "put" requests to update the key-value store (an abstraction layer acting like a giant map) 
or "get" requests to retrieve a value associated with a specific key. 

Regardless of the message type, the server processes the request, performs the appropriate operation on the store, 
and transmits a response message back over the network. [Store](https://github.com/SarthakMakhija/many-flavors-of-networking-io/tree/main/single_thread_blocking_io/store) is an abstraction over Golang's map.

The message format itself includes fields for the key, value, message type ("put" or "get"), and a status code. Status code 
is used in the response messages. `KeyValueMessage` is represented with the following proto format:

```protobuf
message KeyValueMessage {
  string key = 1;
  string value = 2;
  uint32 kind = 3;
  Status status = 4;
}

enum Status {
  Ok = 0;
  NotOk = 1;
}
```

We will build our TCP server(s) in Golang.

Let's start by building a TCP server that is single-threaded and has blocking system calls.

### Single-Threaded Blocking IO

This approach keeps things simple by using a single thread for both listening for connections and handling them. 
When a connection arrives, the thread reads data until the end is reached (`EOF`) or an error occurs.

All the IO operations like accepting new connections: `listener.Accept(..)`, reading from connection: `connection.Read(...)`
and writing to connection: `connection.Write(...)` are blocking.

> Blocking I/O means the thread execution gets stuck waiting for an I/O operation to complete, like reading data from 
> a file or writing to a file. The thread can't do anything else until the I/O operation finishes. 
> 
> This waiting state often leads to a context switch. The operating system temporarily suspends the blocked thread and 
> switches to another ready-to-run thread, maximizing CPU utilization.

**Pros:**
- Easy to understand and implement.

**Cons:**
- **Limited scalability**: Only one client (over one TCP connection) can be served at a time, meaning others have to wait.
- **Resource bottleneck**: A single slow client can stall other clients.
- **Under-utilized hardware**: Modern CPUs can handle many tasks concurrently, but this approach doesn't leverage that.

The idea is implemented using `TCPServer`, `IncomingTCPConnection`, `ConnectionReader` and a custom deserialization method.

Let's start with the `TCPServer`. The following code creates a new instance of `TCPServer` and creates a listener on the specified
host and port.

```go
// NewTCPServer creates a new instance of TCPServer.
func NewTCPServer(host string, port uint16) (*TCPServer, error) {
	address := fmt.Sprintf("%s:%v", host, port)
	listener, err := net.Listen("tcp", address)
	if err != nil {
		return nil, err
	}
	return &TCPServer{
		address:  address,
		listener: listener,
		store:    store.NewInMemoryStore(),
	}, nil
}
```

The `Start` method of `TCPServer` runs in a tight loop waiting for new TCP connections. The `Accept` method is blocking in nature, 
which means the main goroutine running the `Start` is suspended until a new connection arrives.

Once a connection arrives, the server creates a dedicated object (`IncomingTCPConnection`) to handle messages on that 
specific connection. However, this handling still happens within the same main goroutine, meaning the server can only 
focus on one client at a time. This approach is known as "Single-Threaded Blocking IO."

```go
// Start starts the server.
// TCPServer implements "Single thread blocking IO" pattern.
// TCPServer:
// - runs a continuous loop in a single goroutine (/main goroutine).
// - a new instance of IncomingTCPConnection is created for every new connection.
// - the incoming TCP connection is handled in the same main goroutine.
// - this pattern involves blocking IO.
func (server *TCPServer) Start() {
	for {
		connection, err := server.listener.Accept() // Blocks until a connection arrives
		if err != nil {
			return
		}
		conn.NewIncomingTCPConnection(connection, server.store).Handle()
	}
}
```

The `Handle` method manages the incoming connection using an infinite loop.
It attempts to read one message per iteration from the connection using the `AttemptReadOrErrorOut`. 
If any error occurs during reading (including reaching the end of the stream - `io.EOF`), the loop exits.
After a successful read, the code then checks the message type (Kind) to determine how to handle it.

```go
// Handle handles the incoming connection.
// It runs an infinite loop, trying to read from the connection.
// The method AttemptReadOrErrorOut() of ConnectionReader reads from the connection and returns the incoming message or an error.
// The method returns if there is any error (including io.EOF) in reading from the connection.
func (incomingConnection IncomingTCPConnection) Handle() {
	for {
		select {
		case <-incomingConnection.closeChannel:
			return
		default:
			incomingMessage, err := incomingConnection.connectionReader.AttemptReadOrErrorOut()
			if err != nil {
				return
			}
			switch incomingMessage.Kind {
			case proto.KeyValueMessageKindPutOrUpdate:
				incomingConnection.handlePutOrUpdate(incomingMessage)
			case proto.KeyValueMessageKindGet:
				incomingConnection.handleGet(incomingMessage)
			}
		}
	}
}
```

The method `AttemptReadOrErrorOut` handles reading messages from the connection. 
It first checks for a closing signal on the `closeChannel`. If detected, it returns an error indicating a closed connection. 
Otherwise, it sets a read deadline of 20 milliseconds on the underlying connection using `SetReadDeadline`. 
This prevents the program from hanging indefinitely on the blocking read method if no data arrives within the specified time.

> Please note: read system call happens inside `proto.DeserializeFrom`.

The method then attempts to deserialize a message using `proto.DeserializeFrom`. If successful, the message is returned. 

However, if an error occurs, it checks if the error is a timeout. If it's a timeout, the method tracks the number of 
consecutive timeouts (`totalTimeoutsErrors`). As long as the number of timeouts is within a tolerable limit 
(`maxTimeoutErrorsTolerable`), the loop continues to try reading. 
This allows for handling temporary network fluctuations without immediately dropping the connection. 
If the timeout limit is reached or any other non-timeout error occurs, the method returns an error.

```go
// AttemptReadOrErrorOut attempts to read from the incoming TCP connection.
// It runs an infinite loop to read a single message from the incoming connection.
//
// proto.DeserializeFrom() reads from the connection using "blocking IO" and returns either a message or an error.
// The method tolerates network timeout errors.
//
// This method also sets ReadDeadline for future Read calls and any currently-blocked Read call.
func (connectionReader ConnectionReader) AttemptReadOrErrorOut() (*proto.KeyValueMessage, error) {
	totalTimeoutsErrors := 0
	for {
		select {
		case <-connectionReader.closeChannel:
			return nil, errors.New("ConnectionReader is closed")
		default:
			_ = connectionReader.connection.SetReadDeadline(time.Now().Add(20 * time.Millisecond))

			message, err := proto.DeserializeFrom(connectionReader.bufferedReader)
			if err != nil {
				if errors.As(err, &connectionReader.netError) && connectionReader.netError.Timeout() {
					totalTimeoutsErrors += 1
				}
				if totalTimeoutsErrors <= maxTimeoutErrorsTolerable {
					continue
				}
				return nil, err
			}
			return message, nil
		}
	}
}
```

The method `DeserializeFrom` reads `KeyValueMessage` the connection.
`KeyValueMessage` is serialized in the following format: 4 bytes to denote the message size followed by the actual message and then
the footer.

> Please note, the footer was not needed in this implementation. The idea of footer is used in non-blocking with busy wait
> and event loop implementations. To keep the code same, footer was added in all the serialization implementations. 

```go
// DeserializeFrom deserializes the reader into KeyValueMessage.
// Usually the incoming connection is passed as a reader.
func DeserializeFrom(reader io.Reader) (*KeyValueMessage, error) {
	headerBytes := make([]byte, ReservedHeaderLength)
	_, err := reader.Read(headerBytes)
	if err != nil {
		return nil, err
	}

	bodyWithFooter := make([]byte, binary.LittleEndian.Uint32(headerBytes))
	_, err = reader.Read(bodyWithFooter)
	if err != nil {
		return nil, err
	}

	message := &KeyValueMessage{}
	err = proto.Unmarshal(bodyWithFooter[:len(bodyWithFooter)-FooterLength], message)
	if err != nil {
		return nil, err
	}
	return message, nil
}
```

The complete implementation is available [here](https://github.com/SarthakMakhija/many-flavors-of-networking-io/tree/main/single_thread_blocking_io).

### Multi-Threaded Blocking IO

This approach tackles the limitations of single-threaded blocking by introducing multiple threads. 
Instead of a single thread handling everything, a dedicated thread is spawned for each incoming connection. 
This allows the main thread to continue accepting new connections while other threads handle existing ones.

**Pros:**
- Multiple threads can handle clients simultaneously, enhancing responsiveness.

**Cons:**
- Creating and managing threads requires resources, so it's important to find a balance between the number of threads and available resources.

> There's a limit to how many threads a machine can handle effectively. 
> While a CPU-bound application on a 16-core machine might benefit from 16 threads (one per core), I/O-bound applications require a different approach.
> 
> In I/O-bound scenarios, threads can get blocked waiting for data (like reading from a network). 
> To utilize CPU time efficiently, the operating system can swap a blocked thread with another ready-to-run thread. 
> This context switching improves responsiveness, but it's not magic.
> 
> Creating too many threads can become counterproductive. 
> As the number of threads grows, the OS spends more time managing them (context switching) instead of executing actual tasks. 
> This overhead can reduce the application's overall performance.
>
> In the world of concurrency, the "c10k problem" refers to the challenge of efficiently managing 
> around 10,000 concurrent connections, highlighting the trade-off between threads and performance.

The only change is in the `Start` method of the TCPServer which creates a new goroutine for each incoming connection. All the IO operations
are still blocking.

The `Start` method of `TCPServer` runs in a tight loop waiting for new TCP connections. The `Accept` method is blocking in nature,
which means the main goroutine running the `Start` is suspended until a new connection arrives.

Once a connection arrives, the server creates a dedicated object (`IncomingTCPConnection`) to handle messages on that
specific connection. However, this handling happens in a new goroutine and the approach is known as "Multi-Threaded Blocking IO."
This approach follows unbounded concurrency, meaning there's no predefined limit on the number of goroutines the server can spawn.
(Proper profiling and benchmarking should be done to decide if goroutine pooling would help or not).

```go
// Start starts the server.
// TCPServer implements "Multi thread blocking IO" pattern.
// TCPServer:
// - runs a continuous loop in a single goroutine (/main goroutine).
// - a new instance of IncomingTCPConnection is created for every new connection.
// - The incoming TCP connection is handled in new goroutine.
// - This pattern involves goroutine per connection and blocking IO to read from the incoming connection.
func (server *TCPServer) Start() {
	for {
		connection, err := server.listener.Accept()
		if err != nil {
			return
		}
		go conn.NewIncomingTCPConnection(connection, server.store).Handle()
	}
}
```

The complete implementation is available [here](https://github.com/SarthakMakhija/many-flavors-of-networking-io/tree/main/multi_thread_blocking_io).

### Non-blocking with Busy Wait

This approach tackles the limitations of blocking I/O by using non-blocking sockets.
Here, the server sets the socket file descriptor to non-blocking mode with `syscall.SetNonblock(serverFd, true)`. 
This means the server won't get stuck waiting for I/O operations to complete (like reading data) on the `serverFd`.

However, non-blocking comes with a trade-off. If the server attempts to read from a non-blocking socket with no available data, it won't block, but an error 
like `EAGAIN` or `EWOULDBLOCK` will be returned. To address this, the server employs a technique called "busy waiting."

In busy waiting, the server continuously polls the socket (checks for data availability) until it receives an end-of-file 
(`EOF`) signal or encounters another error.  

**Pros:**
- Since the server isn't blocked waiting for I/O operations, it should be more responsive.

**Cons:**
- Busy waiting consumes CPU resources as the server constantly polls the socket for data. This can become a significant issue 
with a large number of connections, potentially leading to performance degradation.
- Handling errors may become tricky. Distinguishing actual errors from those indicating no available data can be challenging in this approach.
My implementation only considers `EAGAIN` or `EWOULDBLOCK` as the errors indicating no available data.

Overall, Non-Blocking with Busy Wait can be a good choice for situations where responsiveness is a priority and the 
number of connections is relatively low. 

Let's jump into the implementation.

The following code creates a new instance of `TCPServer`. It starts by creating a TCP socket using `syscall.Socket`.
- `AF_INET` indicates an IPV4 socket,
- `SOCK_STREAM` indicates a bidirectional sockets and,
- `0` indicates a TCP socket 

The server file descriptor `serverFd` is marked non-blocking using `syscall.SetNonblock`.

Next, the server binds itself to the specified host (IP address) and port using `syscall.Bind`. 
Finally, the socket starts listening for incoming connections with `syscall.Listen`, setting a maximum limit on the number of clients (`MaxClients`).

```go
func NewTCPServer(host string, port uint16) (*TCPServer, error) {
	//starts the listener on the given port and returns the server file descriptor, if there is no error.
	startListener := func() (int, error) {
		// syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0) creates an IPv4 (AF_INET), bidirectional (SOCK_STREAM), TCP (0) socket.
		serverFd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0)
		if err != nil {
			_ = syscall.Close(serverFd)
			return -1, err
		}
		// SetNonblock sets the server file descriptor non-blocking. This means the file descriptor can be polled.
		// A non-blocking file descriptor does not block on IO operations and can be polled.
		if err = syscall.SetNonblock(serverFd, true); err != nil {
			_ = syscall.Close(serverFd)
			return -1, err
		}

		ip4 := net.ParseIP(host)
		if err = syscall.Bind(serverFd, &syscall.SockaddrInet4{
			Port: int(port),
			Addr: [4]byte{ip4[0], ip4[1], ip4[2], ip4[3]},
		}); err != nil {
			_ = syscall.Close(serverFd)
			return -1, err
		}
		if err = syscall.Listen(serverFd, MaxClients); err != nil {
			return -1, err
		}
		return serverFd, nil
	}
	serverFd, err := startListener()
	if err != nil {
		return nil, err
	}

	store := store2.NewInMemoryStore()
	return &TCPServer{
		serverFd: serverFd,
		handlers: map[uint32]conn.Handler{
			proto.KeyValueMessageKindPutOrUpdate: conn.NewPutOrUpdateHandler(store),
			proto.KeyValueMessageKindGet:         conn.NewGetHandler(store),
		},
		stopChannel: make(chan struct{}),
	}, nil
}
```

The following code starts the `TCPServer`. The method runs a loop to continuously check for incoming connections. 
However, unlike blocking I/O, it uses `syscall.Accept` method on the non-blocking file descriptor. 
This means the server won't get stuck waiting for new connections. If `syscall.Accept` doesn't return an error, the code creates a 
new client (an abstraction) to handle the incoming connection. 

> The code only checks for `syscall.EAGAIN` and `syscall.EWOULDBLOCK` errors to indicate that no data is currently available 
> on the socket.

It's important to note that **this implementation** is still **single-threaded**. 
Even though the server does non-blocking I/O, it can only focus on one client at a time due to the nature of the loop. 
This limitation contrasts with approaches that utilize multiple threads or event loops for true parallel processing.

```go
// Start starts the server.
// TCPServer implements "Non-Blocking with Busy-Wait" pattern.
// TCPServer:
// - runs a continuous loop in a single goroutine (/main goroutine).
// - serverFd is already marked non-blocking, this means any IO operations on this file descriptor will not block. However, the file descriptor can be polled.
// - an incoming connection is represented by its file descriptor "connectionFd".
// - connectionFd is also marked non-blocking.
// - a new client is created (for the incoming connectionFd) which handles the connection.
// - all the IO operations are non-blocking.
// This server handles only one client (/connection) at a time.
func (server *TCPServer) Start() {
	for {
		select {
		case <-server.stopChannel:
			return
		default:
			connectionFd, _, err := syscall.Accept(server.serverFd)
			if err != nil {
				if errors.Is(err, syscall.EAGAIN) || errors.Is(err, syscall.EWOULDBLOCK) {
					continue
				}
				return
			}
			_ = syscall.SetNonblock(connectionFd, true)
			conn.NewClient(connectionFd, server.handlers).Run()
		}
	}
}
```

`Client` is a simple abstraction that polls the connection file descriptor. It runs in a tight loop, and attempts to read 
from the socket.

```go
// NewClient creates a new instance of the client.
// It reads chunks from the connection file descriptor and maintains the current buffer.
// currentBuffer denotes the chunk that is read currently.
// The provided file descriptor is set to non-blocking by the caller.
func NewClient(fd int, handlers map[uint32]Handler) *Client {
	return &Client{
		fd:            fd,
		handlers:      handlers,
		stopChannel:   make(chan struct{}),
		readBuffer:    make([]byte, 1024),
		currentBuffer: bytes.NewBuffer([]byte{}),
	}
}

// Run runs the client.
func (client *Client) Run() {
	for {
		select {
		case <-client.stopChannel:
			return
		default:
			keyValueMessage, err := client.read()
			if err != nil {
				return
			}
			if keyValueMessage == nil {
				continue
			}
			if err := client.handle(keyValueMessage); err != nil {
				return
			}
		}
	}
}
```

The method `client.read` reads a single `proto.KeyValueMessage` from the file descriptor in a non-blocking manner. 
Since the file descriptor is already set to non-blocking, the `syscall.Read(...)` call won't wait if no data is immediately 
available. However, an error might be returned in such cases (`EAGAIN` or `EWOULDBLOCK`).

The `read` method employs a loop to handle different scenarios:

- **Successful read**: If `syscall.Read(...)` returns a positive value (`n`) and no errors occur, the received data is appended to the `client.currentBuffer`.
This buffer acts as a temporary storage for incomplete message fragments.
- **No data available**: If `n` is zero, the loop exits, indicating no data was read. 
Errors like `EAGAIN` or `EWOULDBLOCK` also signal this condition and the loop exits without raising an error.
- **End of file (EOF)**: An `io.EOF` error signifies the end of the connection, and the method returns the error.
- **Other errors**: Any other error encountered during the read process terminates the loop and the method returns the error.
- **Finding the message footer**: The loop continues until the `readBuffer` contains the message footer bytes (`proto.FooterBytes`). 
This signifies the complete message has been received.
- **Deserializing the message**: A single instance of `proto.KeyValueMessage` is created from the current buffer.

```go
// read reads a single proto.KeyValueMessage from the file descriptor.
// The file descriptor is already set to non-blocking, which means syscall.Read(..) will not block.
// However, if there is nothing to be read from the file descriptor, an error would be returned.
// The error would be EAGAIN or EWOULDBLOCK.
// For any error, other than EAGAIN or EWOULDBLOCK, the read method will return.
// read will continue reading till it finds the proto.FooterBytes.
// However, it is possible that syscall.Read(..) does not return the amount of data that is requested.
// In that case, the received data will be stored in client.currentBuffer and the read method will perform poll again.
// When it polls again, it will read further data until proto.FooterBytes are found.
// The combined data represented by the currentBuffer will be deserialized into proto.KeyValueMessage.
func (client *Client) read() (*proto.KeyValueMessage, error) {
	for {
		n, err := syscall.Read(client.fd, client.readBuffer)
		if n <= 0 {
			break
		}
		client.currentBuffer.Write(client.readBuffer[:n])
		if err != nil {
			if err == io.EOF {
				return nil, err
			}
			if errors.Is(err, syscall.EAGAIN) || errors.Is(err, syscall.EWOULDBLOCK) {
				return nil, nil
			}
			return nil, err
		}
		if bytes.Contains(client.readBuffer, proto.FooterBytes) {
			break
		}
	}
	if client.currentBuffer.Len() > 0 {
		keyValueMessage, err := proto.DeserializeFrom(client.currentBuffer)
		if err != nil {
			return nil, err
		}
		return keyValueMessage, nil
	}
	return nil, nil
}
```

The complete implementation is available [here](https://github.com/SarthakMakhija/many-flavors-of-networking-io/tree/main/non_blocking_busy_waiting).

### Single-Threaded Event loop

This approach tackles the limitations of blocking I/O by employing **non-blocking sockets** and an **event loop** that leverages **I/O multiplexing**. 
The event loop constantly monitors sockets for incoming data or events (connection requests, disconnections, etc.).

The key ideas include:

- **Registering for Events**: The server uses system calls like `epoll`, `select`, or `kqueue` to register file descriptors of 
sockets with the operating system. These system calls essentially tell the kernel which sockets the server wants to be notified about.
  > Kqueue works on Darwin, epoll, select work on Linux and IOCP on Windows.
- **Kernel's Watchlist**: The operating system maintains a data structure (like kqueue for some systems) to efficiently track the 
registered sockets and any events associated with them. This structure becomes a central point for gathering event information.
- **Polling for Updates**: Instead of actively checking each socket individually, the server periodically "polls" the event 
loop (using the chosen system call). This poll essentially asks the kernel if any events have occurred on the registered sockets.
- **Event Handling**: If kernel detects an event (data arrival, connection request, etc.), it informs the event loop. 
The loop then triggers the appropriate handler function to process the specific event.

**Pros:**
- Compared to blocking I/O, the server doesn't get stuck waiting for data on individual sockets. The event loop allows it to 
remain responsive and handle other events while waiting for data to arrive.
- Polling the event loop for multiple sockets is a more efficient way to manage I/O compared to constantly checking each socket 
individually. This reduces CPU overhead associated with busy waiting.

**Cons:**
- Handling errors may become tricky. The server needs to handle various events (data arrival, connection closure, errors) and 
ensure proper error propagation through the event handlers.

The following code creates a new instance of `TCPServer`. This code is almost same as the one [here](#non-blocking-with-busy-wait).
The only thing that is different here is the creation of event loop using: `NewEventLoop`.

```go
// NewTCPServer creates a new instance of TCPServer.
func NewTCPServer(host string, port uint16) (*TCPServer, error) {
	//starts the listener on the given port and returns the server file descriptor, if there is no error.
	startListener := func() (int, error) {
		// syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0) creates an IPv4 (AF_INET), bidirectional (SOCK_STREAM), TCP (0) socket.
		serverFd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0)
		if err != nil {
			_ = syscall.Close(serverFd)
			return -1, err
		}
		// SetNonblock sets the server file descriptor non-blocking. This means the file descriptor can be polled.
		// A non-blocking file descriptor does not block on IO operations and can be polled.
		if err = syscall.SetNonblock(serverFd, true); err != nil {
			_ = syscall.Close(serverFd)
			return -1, err
		}

		ip4 := net.ParseIP(host)
		if err = syscall.Bind(serverFd, &syscall.SockaddrInet4{
			Port: int(port),
			Addr: [4]byte{ip4[0], ip4[1], ip4[2], ip4[3]},
		}); err != nil {
			_ = syscall.Close(serverFd)
			return -1, err
		}
		if err = syscall.Listen(serverFd, MaxClients); err != nil {
			return -1, err
		}
		return serverFd, nil
	}
	//createEventLoop creates an instance of Event loop.
	createEventLoop := func(serverFd int, store *store.InMemoryStore) (*event_loop.EventLoop, error) {
		eventLoop, err := event_loop.NewEventLoop(serverFd, MaxClients, map[uint32]conn.Handler{
			proto.KeyValueMessageKindPutOrUpdate: conn.NewPutOrUpdateHandler(store),
			proto.KeyValueMessageKindGet:         conn.NewGetHandler(store),
		})
		if err != nil {
			return nil, err
		}
		return eventLoop, nil
	}
	//init creates an instance of TCPServer.
	init := func() (*TCPServer, error) {
		serverFd, err := startListener()
		if err != nil {
			return nil, err
		}
		eventLoop, err := createEventLoop(serverFd, store.NewInMemoryStore())
		if err != nil {
			return nil, err
		}
		return &TCPServer{
			serverFd:  serverFd,
			eventLoop: eventLoop,
		}, nil
	}
	return init()
}
```

Let's look at the `NewEventLoop` method. This method does the following:

- **KQueue Creation**: The method starts by creating a new `KQueue` using `NewKQueue`. This KQueue is a kernel data structure 
that acts like a central hub for tracking events on registered file descriptors (sockets).
- **Server File Descriptor Subscription**: Next, the method subscribes the server's file descriptor (representing the listening socket) to the 
`KQueue` using `subscribeRead`. This essentially tells the kernel that the server wants to be notified whenever there's 
something to read on that socket (typically indicating a new incoming connection).
  - The `EVFILT_READ` filter specifies that the server is interested in the read events. `EVFILT_READ` for server file descriptor
    indicates new connection.

```go
// NewEventLoop creates a new instance of EventLoop.
// It also subscribes using the EVFILT_READ filter on the server file descriptor.
func NewEventLoop(serverFd int, maxClients int, clientHandlers map[uint32]conn.Handler) (*EventLoop, error) {
	// NewKQueue creates a new kernel KQueue data structure to hold various events on the subscribed file descriptor.
	kQueue, err := NewKQueue(maxClients)
	if err != nil {
		return nil, err
	}
	eventLoop := &EventLoop{
		serverFd:       serverFd,
		kQueue:         kQueue,
		clients:        make(map[int]*Client),
		clientHandlers: clientHandlers,
		stopChannel:    make(chan struct{}),
	}
	// subscribes to the given server file descriptor using EVFILT_READ and EV_ADD flag.
	// This means an event will be added to the kernel KQueue when the server file descriptor is ready to be read
	// (/meaning there is an incoming connection on the server).
	err = eventLoop.subscribeRead(serverFd)
	if err != nil {
		return nil, err
	}
	return eventLoop, nil
}
```

The complete implementation of `KQueue` is available [here](https://github.com/SarthakMakhija/many-flavors-of-networking-io/blob/main/single_thread_event_loop/event_loop/kqueue_darwin.go).

The `Start` method of `TCPServer` runs `EventLoop`.

```go
// Start starts the server which in turn starts the event loop.
// TCPServer implements "Single thread Non-Blocking with event loop" pattern.
func (server *TCPServer) Start() {
	server.eventLoop.Run()
}
```

The `Run` method is launched in a separate goroutine to avoid blocking the main thread. Here's what happens within the loop:

- **Polling for Events**: The loop continuously utilizes the `kQueue.Poll` method to retrieve any pending events on the subscribed 
file descriptors. `kQueue.Poll` blocks if there are no events on the registered file descriptors.
- **Handling Events**: The retrieved events are then processed one by one. The code differentiates between two main scenarios:
  - **Server File Descriptor**: If the event's file descriptor matches the server's file descriptor (meaning `event.Ident` equals `eventLoop.serverFd`), 
  it signifies a new incoming connection. The `acceptClient` function is responsible for handling this event and creating 
  a new `Client` object.
  - **Existing Client**: If the event doesn't originate from the server socket, it likely belongs to an existing client 
  connection (identified by `event.Ident`). In this case, the `runClient` function is called to process any data or events 
  associated with that particular client.
  - **Terminating Client**: If the event signifies `EOF`, the corresponding client is stopped.
  
```go
// Run runs an event loop. It:
// - runs an event loop in its own goroutine.
// - polls the KQueue for events on the subscribed file descriptors.
// - if the polled event's file descriptor is same as the server's file descriptor: a new client is accepted,
// - else: an existing client for the file descriptor is run.
func (eventLoop *EventLoop) Run() {
	// TODO: Handle client error
	go func() {
		for {
			select {
			case <-eventLoop.stopChannel:
				return
			default:
				events, err := eventLoop.kQueue.Poll(-1)
				if err != nil {
					continue
				}
				for _, event := range events {
					if event.Flags&syscall.EV_EOF == syscall.EV_EOF {
						eventLoop.stopClient(int(event.Ident))
						delete(eventLoop.clients, int(event.Ident))
						continue
					}
					if int(event.Ident) == eventLoop.serverFd {
						if err := eventLoop.acceptClient(); err != nil {
							continue
						}
					} else {
						eventLoop.runClient(int(event.Ident))
					}
				}
			}
		}
	}()
}
```

The `acceptClient` accepts the new connection by invoking `syscall.Accept`. The system call does not block because the server file
descriptor is marked non-blocking. If `syscall.Accept` doesn't return an error, the code creates a new client (an abstraction) 
to handle the incoming connection. The connection file descriptor is set to non-blocking mode and registered with `KQueue`. 
This means file descriptors of all sockets are registered with `KQueue` and `EventLoop` keeps on polling `KQueue` to check is any
of the file descriptors are ready to be acted upon.

```go
// acceptClient accepts a new client (/socket).
// syscall.Accept(..) will not block because the method is called when the non-blocking file descriptor is ready.
func (eventLoop *EventLoop) acceptClient() error {
	fd, _, err := syscall.Accept(eventLoop.serverFd)
	if err != nil {
		return err
	}

	eventLoop.clients[fd] = NewClient(fd, eventLoop.clientHandlers)
	_ = syscall.SetNonblock(fd, true)

	if err := eventLoop.subscribeRead(fd); err != nil {
		return err
	}
	return nil
}
```

The `Client.Run(..)` is similar to what we have already seen, except that it is invoked when the corresponding file descriptor
is ready to be read.

> It's important to consider that a single read operation might not always retrieve the complete `KeyValueMessage` data.
> This can happen if the received data doesn't reach the message boundary (marked by `proto.FooterBytes`).
> 
> The received data is stored in the `client.currentBuffer`. This buffer acts as a temporary storage for incomplete message fragments.
> The `read` method returns control without raising an error. This allows the event loop to handle other events while
> waiting for more data on the socket.
> 
> When the file descriptor becomes ready again (signaling more data is available), the `read` method will be invoked again.
> This process continues until the complete message, identified by the `proto.FooterBytes`, is received.
> 
> Once all parts of the message are accumulated in the `client.currentBuffer`, the combined data is deserialized into a 
> complete `proto.KeyValueMessage` object.

```go
// Run runs the client.
// It is invoked when the client's file descriptor is ready to be read.
func (client *Client) Run() {
	for {
		select {
		case <-client.stopChannel:
			return
		default:
			keyValueMessage, err := client.read()
			if err != nil {
				return
			}
			if err := client.handle(keyValueMessage); err != nil {
				return
			}
		}
	}
}

// read reads a single proto.KeyValueMessage from the file descriptor.
// read will be triggered when the non-blocking file descriptor is ready.
// This means syscall.Read(..) will not block.
// read will continue reading till it finds the proto.FooterBytes.
// However, it is possible that syscall.Read(..) does not return the amount of data that is requested.
// In that case, the received data will be stored in client.currentBuffer and the read method will return.
// When the read method is invoked again, at a later point in time when the file descriptor is ready,
// it will read further data until proto.FooterBytes are found.
// The combined data represented by the currentBuffer will be deserialized into proto.KeyValueMessage.
func (client *Client) read() (*proto.KeyValueMessage, error) {
	for {
		n, err := syscall.Read(client.fd, client.readBuffer)
		if n <= 0 {
			break
		}
		client.currentBuffer.Write(client.readBuffer[:n])
		if err != nil {
			if err == io.EOF {
				break
			}
			return nil, err
		}
		if bytes.Contains(client.readBuffer, proto.FooterBytes) {
			break
		}
	}
	keyValueMessage, err := proto.DeserializeFrom(client.currentBuffer)
	if err != nil {
		return nil, err
	}
	return keyValueMessage, nil
}
```

### Mentions

- [Google Bard](https://bard.google.com/chat) helped with the article.

### References

- [Grokking Concurrency](https://www.manning.com/books/grokking-concurrency)
- [DiceDB](https://github.com/DiceDB/dice)