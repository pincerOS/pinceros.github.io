---
title: Sockets
nav_order: 4
parent: Networking
---

# Sockets

This module defines a minimal socket abstraction for a custom `no_std` networking stack, supporting UDP and TCP over IPv4. It's designed to be lightweight and embedded-friendly, leveraging async operations and simple data structures like SPSC queues for buffering.

---

## `SocketAddr`

A `SocketAddr` identifies a network endpoint (IPv4 address + port):

```rust
#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)]
pub struct SocketAddr {
    pub addr: Ipv4Address,
    pub port: u16,
}
```

### DNS Resolution

`SocketAddr::resolve` uses a UDP socket to send a DNS query and waits for a response:

```rust
pub async fn resolve(host: &str, port: u16) -> Self
```

Internally, this:

* Binds a new UDP socket to port 53.
* Sends a serialized DNS query to the configured DNS server.
* Receives and parses the response to extract the resolved `Ipv4Address`.

---

## Socket Lifecycle Operations

All socket file descriptors (`socketfd`) are managed via a global interface's `sockets` map.

### Ephemeral Port Management

An `AtomicU16` is used to track the next ephemeral port:

```rust
pub static NEXT_EPHEMERAL: AtomicU16 = AtomicU16::new(32768);
```

---

### `send_to`

Queues a packet to be sent from a socket:

```rust
pub async fn send_to(socketfd: u16, payload: Vec<u8>, saddr: SocketAddr)
```

1. Verifies socket existence.
2. Auto-binds if not already bound.
3. Enqueues the packet for transmission.

---

### `recv_from`

Blocking receive from a socket’s receive queue:

```rust
pub async fn recv_from(socketfd: u16) -> Result<(Vec<u8>, SocketAddr)>
```

---

### `connect`, `listen`, `accept`, `close`

These follow conventional TCP socket behavior but are stubbed out for UDP:

```rust
pub async fn connect(socketfd: u16, saddr: SocketAddr)
pub async fn listen(socketfd: u16, num_requests: usize)
pub async fn accept(socketfd: u16) -> Result<u16>
pub async fn close(socketfd: u16)
```

---

## Internal Representation: `TaggedSocket`

All sockets are wrapped in an enum that tags their type:

```rust
pub enum TaggedSocket {
    Udp(UdpSocket),
    Tcp(TcpSocket),
}
```

Each variant implements methods like:

* `bind(port)`
* `recv()` / `send_enqueue(payload, saddr)`
* `listen()` / `accept()` / `connect()`
* `close()`

---

## SPSC Queues for Asynchronous Communication

Internally, sockets use **SPSC (Single-Producer Single-Consumer) ring buffers** from `crate::ringbuffer` to implement async send/receive queues.

Each socket has:

* A `Sender<BUFFER_LEN, (Vec<u8>, SocketAddr)>` for the send queue.
* A `Receiver<BUFFER_LEN, (Vec<u8>, SocketAddr)>` for the receive queue.

This avoids the need for locking and is ideal for interrupt-safe or cooperative multitasking environments. In the future, this should be expanded to use a MPSC queue.

---

## Error Handling

All socket functions return a `Result<T>` with custom `Error` types such as:

* `Error::InvalidSocket(socketfd)`
* `Error::BindingInUse(addr)`
* `Error::Ignored` (for operations like `connect` on UDP)

---

## Example Flow

Here’s how a typical TCP/HTTP send might look:

```rust
let host = "neverssl.com";
let saddr = SocketAddr::resolve(host, 80).await;

let s = TcpSocket::new();
match connect(s, saddr).await {
    Ok(_) => println!("initiated connection"),
    Err(_) => println!("couldn't connect to address"),
};

let path = "/";
let http_req = HttpPacket::new(HttpMethod::Get, host, path);
match send_to(s, http_req.serialize(), saddr).await {
    Ok(_) => println!("sent http request"),
    Err(_) => println!("couldn't send"),
};

let (resp, sender) = recv_from(s).await.unwrap();

close(s).await;
```

---

## Binding Behavior

Binding a socket will:

* Fail if another socket is already bound to the same `SocketAddr`.
* Succeed otherwise and update the socket's internal bind state.
* An ephemeral socket will be used if a bind is needed and none has been provided.
