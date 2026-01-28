[![crates.io](https://img.shields.io/crates/v/squiche.svg)](https://crates.io/crates/squiche)
[![docs.rs](https://docs.rs/squiche/badge.svg)](https://docs.rs/squiche)
[![license](https://img.shields.io/github/license/Anapaya/squiche.svg)](https://opensource.org/licenses/BSD-2-Clause)

This is a fork of Cloudflare's [quiche] to support QUIC using [SCION] as the
underlying tranport protocol using the SCION endhost software development kit
([scion-sdk]).

[quiche]: https://docs.quic.tech/quiche/
[SCION]: https://www.scion.org/
[scion-sdk]: https://github.com/Anapaya/scion-sdk

### Configuring connections

The first step in establishing a QUIC connection using squiche is creating a
[`Config`] object:

```rust
let mut config = squiche::Config::new(squiche::SCION_PROTOCOL_VERSION)?;
config.set_application_protos(&[b"example-proto"]);

// Additional configuration specific to application and use case...
```

### Connection setup

On the client-side the [`connect()`] utility function can be used to create
a new connection, while [`accept()`] is for servers:

```rust
// Client connection.
let conn = squiche::connect(Some(&server_name), &scid, local, peer, &mut config)?;

// Server connection.
let conn = squiche::accept(&scid, None, local, peer, &mut config)?;
```

### Handling incoming packets (UDP SCION socket)

Using the connection's [`recv()`] method the application can process
incoming packets that belong to that connection from the network:

```rust
// UDP SCION socket
let to = socket.local_addr().local_addr().unwrap();

loop {
    let (read, from) = socket.recv_from(&mut buf).unwrap();

    let recv_info = squiche::RecvInfo { from, to };

    let read = match conn.recv(&mut buf[..read], recv_info) {
        Ok(v) => v,

        Err(e) => {
            // An error occurred, handle it.
            break;
        },
    };
}
```

### Generating outgoing packets (UDP SCION socket)

Outgoing packet are generated using the connection's [`send()`] method
instead:

```rust
loop {
    let (write, send_info) = match conn.send(&mut out) {
        Ok(v) => v,

        Err(quiche::Error::Done) => {
            // Done writing.
            break;
        },

        Err(e) => {
            // An error occurred, handle it.
            break;
        },
    };

    let dst = scion_proto::address::SocketAddr::from_std(remote_isd_as, send_info.to)
    socket.send_to(&out[..write], dst).unwrap();
}
```

When packets are sent, the application is responsible for maintaining a
timer to react to time-based connection events. The timer expiration can be
obtained using the connection's [`timeout()`] method.

```rust
let timeout = conn.timeout();
```

The application is responsible for providing a timer implementation, which
can be specific to the operating system or networking framework used. When
a timer expires, the connection's [`on_timeout()`] method should be called,
after which additional packets might need to be sent on the network:

```rust
// Timeout expired, handle it.
conn.on_timeout();

// Send more packets as needed after timeout.
loop {
    let (write, send_info) = match conn.send(&mut out) {
        Ok(v) => v,

        Err(quiche::Error::Done) => {
            // Done writing.
            break;
        },

        Err(e) => {
            // An error occurred, handle it.
            break;
        },
    };

    socket.send_to(&out[..write], &send_info.to).unwrap();
}
```

### HTTP/3

Have a look at the [examples] for more complete examples on how to use the
squiche API. In particular, the [UDP SCION socket] example demonstrates the
usage of squiche using SCION as the underlying transport protocol.

[examples]: quiche/examples/
[UDP SCION socket]: quiche/examples/scion-http3-client.rs
