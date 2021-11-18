# Adding a Virtual TLS Interface

Chances are that the software project that you are using is heavily integrated
with OpenSSL. The API's to use OpenSSL are a little bit arcane and hard to use.

Here's an example call to read some encrypted bytes from a socket:

```c
n = SSL_read(conn->ssl, ptr, len);
```

where `ptr` is a pointer to a buffer, `len` is the number of bytes you want to
read, and `conn->ssl` is a `*SSL` struct provided by OpenSSL, that has a ton of
fields on it. `n` is either the number of bytes returned, or an error if it is
less than or equal to 0.

If you tried to grep the OpenSSL source code for a function named `SSL_read`,
you would not find it. OpenSSL uses a ton of macros to implement its
functionality, that get unpacked at build time into specific functions for a
specific architecture. This abstraction makes it more difficult for people to
figure out what code is actually running, and subsequently to determine whether
it is implemented securely or not.

So the first step we are going to want to take is to add a layer of abstraction
between OpenSSL and the rest of the codebase you are working with. Once an
abstraction layer exists, you can add a rustls implementation that satisfies the
same interface that you've added.

Throughout this example, we are going to take this opportunity to replace the
word "SSL", which refers to the deprecated SSLv2 and SSLv3 protocols, with the
word "TLS", which is the protocol that you are actually negotiating and/or want
to negotiate.

### Step 1: Consolidate OpenSSL calls

If OpenSSL calls or the `*SSL` struct initialization happens in multiple places
over the codebase, you want to consolidate those down into a single file or a
single directory. So anywhere you are calling OpenSSL functions directly, you
want to instead call a function in your new file, that gives back an OpenSSL
object, or invokes the right OpenSSL API.

### Step 2: Don't hand out OpenSSL objects

Instead of having code in your new file that returns or directly operates on
OpenSSL objects, like this:

```c
*SSL init_ssl() {
    SSL_CTX    *ctx;
    ctx = SSL_CTX_new(...)
    // Other options set here.
    return SSL_new(ctx);
}

ssize_t
openssl_read(SSL *ssl, void *ptr, size_t len) {
    return SSL_read(ssl, ptr, len);
}
```

Introduce a struct abstraction that you can operate on instead.

```c
struct tls_conn {
	SSL		   *ssl;			/* SSL status, if have SSL connection */
}
```

And then refactor your existing API's to use this object.

```
*tls_conn init_ssl() {
    tls_conn *conn;
    SSL_CTX    *ctx;
    ctx = SSL_CTX_new(...)
    conn = malloc(sizeof tls_conn);
    // Other options set here.
    conn->ssl = SSL_new(ctx);
    return conn;
}

ssize_t
openssl_read(tls_conn *conn, void *ptr, size_t len) {
    return SSL_read(conn->ssl, ptr, len);
}
```

### Step 3: Don't put "openssl" in function names

Now that you have your own object, don't have function names that refer to
OpenSSL at all. In our case, change `openssl_read` to something like `tls_read`.
This is also a chance to change the API's to something that makes sense.

### Step 4: Declare the new function names in a new file

Eventually, we are going to hide these OpenSSL-specific functions behind
build flags, so they won't be called. To ensure the functions still exist and
can be called, put the function definitions in a header file that's separate
from any OpenSSL specific code (maybe `tls-common.h`).

```c
/*
 * Initialize the tls_conn struct with a set of options.
 */
*tls_conn tls_init(...);

/*
 * Read data from a TLS connection into the buffer pointed to by *ptr.
 */
ssize_t tls_read(tls_conn *conn, void *ptr, size_t len);
```

### Some examples you can look at

Curl [uses a `vtls` interface][vtls] to switch between several different TLS
implementations, choosing one at runtime. [Here is how rustls implements that
interface][curl-rustls].

Postgres uses an interface based on [several calls named `pgtls_*`][pgtls] that
are implemented by OpenSSL in a separate file.

[vtls]: https://github.com/curl/curl/blob/master/lib/vtls/vtls.h#L37-L92
[curl-rustls]: https://github.com/curl/curl/blob/master/lib/vtls/rustls.c#L554-L581
[pgtls]: https://github.com/postgres/postgres/blob/master/src/interfaces/libpq/libpq-int.h#L717-L800

### That's it!

Once you've done this, you should have an interface that you could call with
any backend, not just OpenSSL. In the next tutorial we'll cover how to change
compile flags to compile in your new library.
