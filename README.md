# Multithreaded HTTP/1.1 Web Server in C++

A lightweight **multithreaded HTTP/1.1 web server built from scratch in modern C++**, using POSIX TCP sockets and a custom thread pool.

The project focuses on understanding the systems underneath a web server rather than relying on high-level networking frameworks. It implements the complete request path from accepting a TCP connection to parsing an HTTP request and returning a static file.

## Overview

When a browser or HTTP client connects to the server, the request follows this pipeline:

```text
                    Client
                      │
                      │ TCP Connection
                      ▼
              ┌─────────────────┐
              │  Listener Thread│
              │     accept()    │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   Task Queue    │
              │  Thread-Safe    │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     ┌─────────┐  ┌─────────┐  ┌─────────┐
     │ Worker  │  │ Worker  │  │ Worker  │
     │ Thread 1│  │ Thread 2│  │ Thread 3│
     └────┬────┘  └────┬────┘  └────┬────┘
          └────────────┼────────────┘
                       ▼
              ┌─────────────────┐
              │  HTTP Parser    │
              └────────┬────────┘
                       ▼
              ┌─────────────────┐
              │ Static File     │
              │ Handler         │
              └────────┬────────┘
                       ▼
              ┌─────────────────┐
              │ HTTP Response   │
              └────────┬────────┘
                       ▼
                     Client
```

## Key Features

* TCP socket server using POSIX sockets
* HTTP/1.1 request parsing
* `GET` request handling
* Static-file serving
* MIME-type resolution
* HTTP status and error responses
* Fixed-size custom thread pool
* Thread-safe task queue
* Synchronization using `std::mutex` and `std::condition_variable`
* Concurrent handling of multiple client connections
* Modern C++ resource management
* Testing using browsers and command-line HTTP clients

## Why This Project?

Most web applications are built on top of high-level frameworks and runtimes that hide networking and concurrency details.

This project works closer to the operating-system level to understand:

* How TCP servers accept incoming connections
* How data is received and transmitted through sockets
* How raw HTTP requests are represented as byte streams
* How HTTP requests are parsed
* How multiple connections can be processed concurrently
* How threads communicate through a shared task queue
* Why synchronization is required in concurrent programs
* How static files are mapped to HTTP responses
* How server performance changes with different levels of concurrency

The goal is not to replace production servers such as Nginx or Apache, but to understand the fundamental components that make a network server work.

## Architecture

### 1. Listener Thread

The main thread is responsible for accepting incoming TCP connections.

```text
socket()
   ↓
bind()
   ↓
listen()
   ↓
accept()
```

After accepting a connection, the listener places the connection into the shared task queue and immediately returns to accepting new connections.

### 2. Thread-Safe Task Queue

The task queue stores accepted client connections waiting to be processed.

Multiple worker threads may access the queue simultaneously, so access is protected using:

```cpp
std::mutex
std::condition_variable
```

Workers sleep when the queue is empty and are notified when a new connection becomes available.

### 3. Worker Thread Pool

A fixed number of worker threads are created when the server starts.

Each worker:

1. Waits for a task.
2. Retrieves a client connection from the queue.
3. Receives the HTTP request.
4. Parses the request.
5. Locates the requested file.
6. Generates an HTTP response.
7. Sends the response to the client.
8. Closes the connection.
9. Waits for the next task.

This avoids creating a new operating-system thread for every connection.

### 4. HTTP Request Parser

The server processes raw HTTP request data such as:

```http
GET /index.html HTTP/1.1
Host: localhost:8080
User-Agent: curl/8.0
```

The parser extracts information such as:

```text
Method  → GET
Path    → /index.html
Version → HTTP/1.1
Headers → Host, User-Agent, ...
```

### 5. Static File Handler

Requested paths are mapped to files inside the server's document root.

For example:

```text
/index.html  → public/index.html
/style.css   → public/style.css
/app.js      → public/app.js
```

The server determines an appropriate MIME type and constructs the HTTP response.

Example:

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 248

<html>
    <body>
        <h1>Hello from C++</h1>
    </body>
</html>
```

## Technologies

| Component       | Technology                              |
| --------------- | --------------------------------------- |
| Language        | C++20                                   |
| Networking      | POSIX TCP sockets                       |
| Concurrency     | `std::thread`                           |
| Synchronization | `std::mutex`, `std::condition_variable` |
| Task Management | `std::queue`                            |
| Build System    | CMake                                   |
| Compiler        | GCC / Clang                             |
| Platform        | Linux / Ubuntu                          |

## Project Structure

```text
multithreaded-http-server/
├── CMakeLists.txt
├── README.md
├── include/
│   ├── server.hpp
│   ├── thread_pool.hpp
│   ├── http_parser.hpp
│   └── http_response.hpp
├── src/
│   ├── main.cpp
│   ├── server.cpp
│   ├── thread_pool.cpp
│   ├── http_parser.cpp
│   └── http_response.cpp
├── public/
│   ├── index.html
│   ├── style.css
│   └── ...
└── tests/
    └── ...
```

The structure may evolve as the implementation grows.

## Build

Clone the repository:

```bash
git clone https://github.com/Deviprasadvkk/multithreaded-http-server.git
cd multithreaded-http-server
```

Create a build directory:

```bash
mkdir build
cd build
```

Configure the project:

```bash
cmake ..
```

Compile:

```bash
cmake --build .
```

## Run

Start the server:

```bash
./http_server
```

The server listens on the configured port, for example:

```text
Server started on port 8080
Listening for connections...
```

Open the server in a browser:

```text
http://localhost:8080
```

You can also test it using `curl`:

```bash
curl http://localhost:8080/
```

Or request a specific file:

```bash
curl http://localhost:8080/index.html
```

## Example Request Flow

A request such as:

```http
GET /index.html HTTP/1.1
Host: localhost:8080
```

is processed as:

```text
TCP connection
      ↓
accept()
      ↓
Task queue
      ↓
Worker thread
      ↓
Receive HTTP request
      ↓
Parse method + path
      ↓
Locate index.html
      ↓
Read file
      ↓
Build HTTP response
      ↓
send()
      ↓
Client
```

## Concurrency Model

The server uses a fixed-size thread pool instead of creating a new thread for every incoming connection.

```text
                 Incoming Connections
                         │
                         ▼
                  Listener Thread
                         │
                         ▼
                  ┌─────────────┐
                  │ Task Queue  │
                  └──────┬──────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
        Worker 1      Worker 2      Worker 3
            │            │            │
            └────────────┼────────────┘
                         ▼
                   HTTP Handling
```

The queue is protected against concurrent access, while `std::condition_variable` allows workers to sleep when there are no pending connections instead of continuously polling the queue.

## Error Handling

The server handles common HTTP errors such as:

```text
400 Bad Request
404 Not Found
405 Method Not Allowed
500 Internal Server Error
```

The implementation also handles invalid requests and client-side connection failures where appropriate.

## Learning Objectives

This project is designed to strengthen understanding of:

* TCP/IP networking
* Socket programming
* HTTP/1.1
* Client-server architecture
* C++ multithreading
* Thread synchronization
* Mutexes and condition variables
* Producer-consumer patterns
* Operating-system thread management
* File I/O
* Resource management
* Network-server architecture
* Performance testing

## Testing and Benchmarking

Concurrent request handling will be tested under different worker-pool configurations.

Areas of interest include:

```text
Throughput
Latency
Concurrent connections
Worker count
Queue contention
CPU utilization
```

Benchmark results will be added once measured experimentally.

## Design Trade-offs

### Thread Pool vs. Thread per Connection

A thread-per-connection model creates a new operating-system thread for every incoming connection.

This project instead uses a fixed worker pool:

```text
Connection 1 ──┐
Connection 2 ──┤
Connection 3 ──┼──► Task Queue ──► Fixed Worker Pool
Connection 4 ──┤
Connection 5 ──┘
```

This limits the number of active worker threads and avoids repeated thread creation for each connection.

### Why C++?

C++ provides direct access to:

* Operating-system networking APIs
* Native threads
* Synchronization primitives
* Explicit resource-management patterns
* Low-level I/O behavior

This makes it possible to study the underlying server architecture rather than relying entirely on framework abstractions.

## Current Scope

The current project focuses on:

```text
HTTP/1.1
GET requests
Static files
TCP networking
Custom thread pool
Thread-safe task queue
Concurrent request handling
```

Advanced protocols and features such as HTTP/2, HTTP/3, TLS, WebSockets, databases, and reverse-proxy functionality are intentionally outside the scope of this project.

## Future Improvements

Potential improvements within the current systems/server scope include:

* Graceful server shutdown
* Better request validation
* Connection timeouts
* Improved HTTP parsing
* Additional HTTP status handling
* More comprehensive test coverage
* Performance profiling
* Load testing under different concurrency levels

## Motivation

This project was built to move beyond high-level web frameworks and understand the lower-level mechanisms behind HTTP servers, networking, and concurrent request processing.

The implementation emphasizes **understanding the architecture and trade-offs**, not simply producing a working web page.

## License

This project is intended for educational and portfolio purposes.
