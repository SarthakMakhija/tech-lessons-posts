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

### Overview of TCP Server

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

### Multi-Threaded Blocking IO

### c10k

### Non-blocking with Busy Wait

### Single-Threaded Event loop

### Mentions

### References