# TLS Connection Lifecycle

In this post we'll walk through the lifecycle of a TLS connection and where
rustls gets involved at each step of the process.

It's important to note what rustls-ffi will and won't do for you. `rustls-ffi`
will not:

- Open a socket
- Read or write data to or from the socket
- Load certificates from disk or from the system trust store.

You are responsible for all of those pieces. `rustls-ffi` has helpers for each,
but you have to do them. What `rustls-ffi` will help with is:

- Parsing TLS packets
- Negotiating a TLS version and ciphersuite
- Verifying whether a server validates against the provided TLS certificates
- Encrypting and decrypting TLS data.

The way that this works is, when you want to write data to the socket:

1. You hand the unencrypted data to rustls, via `rustls_connection_write`
2. Rustls encrypts it internally.
3. You invoke `rustls_connection_write_tls`, which invokes a callback with the _encrypted_ data.
4. In the callback you provide, you take encrypted data that Rustls gives you,
and write it to the socket.

When you want to read data, you do this in reverse:

1. You tell Rustls you want to read some data, via `rustls_connection_read_tls`.
2. Rustls invokes a callback that you provide.
3. In the callback, you call `recv` on the socket, to receive the raw encrypted
data.
4. You call `rustls_connection_read` to retrieve the decrypted bytes directly
from Rustls.

## TLS on the Client Device

Now, let's walk through an entire TLS request from the perspective of the
client, and see how this works.

#### Before the request starts

You need to open a connection to the remote server using the `connect` syscall.

You also need to establish the `rustls_connection` object. To do this you want
to:

- create a client config builder (`rustls_client_config_builder_new`)
- set options on it. Any `rustls-ffi` API that accepts a
`client_config_builder` could be called here, for example
`rustls_client_config_builder_load_roots_from_file`).
- exchange it for a client config (`rustls_client_config_builder_build`).

### TLS Handshake

The next step is to initiate a TLS handshake. This process has lots of reads and
writes to the socket, but _no reads and writes of user data_.

You can initiate a TLS handshake by **calling `rustls_connection_write` with a
zero length buffer.** Rustls will prepare the first message to be sent out - a
`ClientHello` message.

To process the `ClientHello` and the rest of the TLS handshake, you are going
to **alternate reading and writing data from the socket until the handshake is
complete.**

#### ClientHello

This is the first message that gets sent by the client - inside will be a list
of algorithms supported by the client, and which TLS protocols we know how to
negotiate.

To actually send the `ClientHello,` you will want to:

- Verify you've called `rustls_connection_write` with an empty buffer, which we
  did above.
- ensure that `rustls_connection_wants_write` returns true (Rustls expects you
  to write some data to the socket).
- call `rustls_connection_write_tls` to tell Rustls you are ready to write data
  to the socket.
- in the callback from `write_tls`, you will want to write the ClientHello
(which will be in a buffer that Rustls will provide for you) to the socket.

You can view example write callbacks here:

- curl: https://github.com/curl/curl/blob/master/lib/vtls/rustls.c#L91-L100
- rustls examples: https://github.com/rustls/rustls-ffi/blob/main/tests/common.c#L120-L134

#### ServerHello + Certificate

The server will send back a `ServerHello` with TLS versions that it supports, as
well as the public certificate that it's going to use to encrypt the data.

To read the `ServerHello`, you will want to do the following:

- Check that `rustls_connection_wants_read` returns true
- call `rustls_connection_read_tls` to read data from the socket.
- If this does not return an error, call `rustls_connection_process_new_packets` to parse the
data you just received.
- If _that_ does not return an error, call `rustls_connection_read` to read the
  data out of the socket.

You can view example read callbacks here:

- curl: https://github.com/curl/curl/blob/master/lib/vtls/rustls.c#L80-L89
- rustls examples: https://github.com/rustls/rustls-ffi/blob/main/tests/common.c#L105-L118

#### Completing the Handshake

We have given examples of how to read data and how to write data. To complete
the handshake, you will want to alternate reads and writes on the socket, until
the socket is ready.

When the handshake is complete, `rustls_connection_is_handshaking` will return
false. This is your signal that you're ready to write some of your own data to
the socket!

#### Wrapping it Up

At a high level, here's what we're doing to set up the handshake, in pseudocode,
and with error checking omitted. If you want a working example, check out the
`tests/client.c` file in the `rustls-ffi` repository.

```c
builder = rustls_client_config_builder_new();
rustls_client_config_builder_load_roots_from_file(builder, "/etc/path/to/certificates")
client_config = rustls_client_config_builder_build(builder);
sock = connect("hostname", "port");
struct rustls_connection *rconn = NULL;
rustls_client_connection_new(builder, "hostname", &rconn);

// Trigger the construction of the ClientHello message and initial handshake.
rustls_connection_write(rconn, "")

for {
    if !rustls_connection_is_handshaking(rconn) {
        // We're done!
        break
    }
    if rustls_connection_wants_read(rconn) {
        // read_cb reads from the socket
        err = rustls_connection_read_tls(rconn, read_cb)
        if (err == EAGAIN || err == EWOULDBLOCK) {
            // Still waiting on the server
            continue
        }
        rustls_connection_process_new_packets(rconn)
        // buf is your buffer for reading data out; we're not expecting
        // anything, though, because this is the handshake.
        rustls_connection_read(rconn, buf)
    }
    if rustls_connection_wants_write(rconn) {
        // write_cb writes to the socket
        rustls_connection_write_tls(rconn, write_cb)
    }
}
```

After this you should be ready to read and write data.

### Writing Data

Now we need to send some bytes, encrypting and decrypting along the way. We need
to trigger an initial write by invoking `rustls_connection_write` with some data
that we want `rustls` to encrypt for us, and we need to read the data from the
server back out by invoking `rustls_connection_read`.

So you could imagine a second for loop, pretty similar to the first.

```c
// Send some data, whatever we want to write on the socket.
// You may need to invoke this more than once dependending on the length of the
// data you want to write, and how much rustls reports has been written.
rustls_connection_write(rconn, "GET / HTTP/1.1\r\n...")

while rustls_connection_wants_write(rconn) {
    // write_cb writes to the socket
    rustls_connection_write_tls(rconn, write_cb)
}
```

### Reading Data

If you are expecting the server to send data, you need to read it back out.

```c
// read_cb is the callback you provide that reads from the socket.
rustls_connection_read_tls(rconn, read_cb)
rustls_connection_process_new_packets(rconn);

// `buf` is your buffer,
// `len` is the size of your buffer
plain_bytes_copied = 0
nbytes = 0
while (plain_bytes_copied < len) {
    result = rustls_connection_read(rconn, buf+plain_bytes_copied,
        len - plain_bytes_copied, &nbytes)
    if (nbytes == 0) {
        // No more bytes coming.
        break
    } else if (result != RUSTLS_RESULT_OK) {
        // handle error
    } else {
        plain_bytes_copied += nbytes;
        nbytes = 0;
    }
}
```

### That's it!

You've got everything you need. To view it all together, you might want to look
at some working implementations.

- curl: https://github.com/curl/curl/blob/master/lib/vtls/rustls.c
- rustls-ffi example code: https://github.com/rustls/rustls-ffi/blob/main/tests/client.c

## TLS on the Server Device

TODO!
