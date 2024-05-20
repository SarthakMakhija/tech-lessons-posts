---
author: "Sarthak Makhija"
title: "Many flavors of Networking IO"
date: 2024-05-20
description: "
The foundation of any networked application hinges on its ability to efficiently handle data exchange. 
But beneath the surface, there's a hidden world of techniques for managing this communication. 
This article dives into the various \"flavors\" of networking IO, exploring the trade-offs associated with each approach. 
We'll delve into techniques like Single-Threaded Blocking IO, Multi-Threaded Blocking IO, Non-blocking with Busy Wait, and Single-Threaded Event loop, 
equipping you with a deeper understanding of how applications juggle network communication with other tasks.
"
tags: ["TCP", "Networking", "Golang", "Event loop"]
thumbnail: "/flavors-of-networking-title.png"
caption: "Background by Bruno Thethe on Pexels"
---

### Introduction

The foundation of any networked application hinges on its ability to efficiently handle data exchange.
But beneath the surface, there's a hidden world of techniques for managing this communication.
This article dives into the various "flavors" of networking IO, exploring the trade-offs associated with each approach.

To illustrate various ways applications handle network traffic, we'll build a TCP server using four distinct approaches: 
**blocking I/O with a single thread**, **blocking I/O with multiple threads**, **non-blocking I/O with busy waiting**, and 
**a single-threaded event loop**. 

Each approach offers unique advantages and drawbacks, and by constructing a server for each approach, we'll gain a deeper 
understanding of their strengths and weaknesses. 

Throughout this exploration, we'll delve into concepts like **blocking vs non-blocking operations**, **threading strategies**, 
and **event loops**, equipping you to choose the most suitable approach for your specific networking needs.

### Overview of our TCP Server

This TCP server operates on messages encoded using the [protobuf](https://protobuf.dev/) format, specifically the `KeyValueMessage`. 
These messages can be either "put" requests to update the key-value store (an abstraction layer acting like a giant map) 
or "get" requests to retrieve a value associated with a specific key. 

Regardless of the message type, the server processes the request, performs the appropriate operation on the store, 
and transmits a response message back over the network. 

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

Let's start by building a TCP server that is single-threaded and has blocking system calls.

### Single-Threaded Blocking IO

This approach keeps things simple by using a single thread for both listening for connections and handling them. 
When a connection arrives, the thread reads data until the end is reached (`EOF`) or an error occurs.

All the IO operations like accepting new connections: `listener.Accept(..)`, reading from connection: `connection.Read(...)`
and writing to connection: `connection.Write(...)` are blocking.

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

### Single-Threaded Event loop

### Mentions

### References