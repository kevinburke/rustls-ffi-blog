# Replacing OpenSSL with rustls

TLS is a critical part of any modern software architecture. Unfortunately, most
TLS libraries are written in languages that are vulnerable to errors like
use-after-free or out of bounds reads.

OpenSSL is maybe the most common library in the world for terminating TLS, on
both the client and the server. It has had a number of security defects that
would not have been possible in a memory safe language that is less vulnerable
to undefined behavior. While OpenSSL has become more secure over time, advances
in the TLS spec - TLS 1.3, for example - mean that new C code is continually
being added to these libraries, which means they are continually at risk of
memory safety errors.

In this series we will walk through how to replace OpenSSL in a given software
installation. We will show you how to use `rustls`, a high performance TLS
library written in Rust, instead of OpenSSL. This series is targeted at people
who do not have specific experience with C, with Rust, or with TLS; all you need
are patience and a keen eye!

The basic idea is to make it possible for the library to compile in either
OpenSSL or rustls based on compilation flags, then work to build compatibility
with rustls, then hopefully work with the team to make rustls the default.

We'll walk through this in the different blog posts in this series:

- Introducing a virtual TLS interface
- Editing compilation flags
- Lifecycle of a TLS request, and how it translates into `rustls-ffi` calls
- Handling errors
- Updating the test suite
- Strategies for getting your PR looked at / merged
- C for people who haven't written C before
