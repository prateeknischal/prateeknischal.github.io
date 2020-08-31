+++
title = "Apache thrift over unix sockets in Rust"
description = "IPC over thrift via unix domain sockets"
date = 2020-08-01
+++

[Apache Thrift](https://en.wikipedia.org/wiki/Apache_Thrift) is an interace
defintion language and binary communication protocol use for defining and
creating services for all the numerous supported languages. It forms an RPC
framework avoiding the usual microservices HTTP style messaging making it a
bit more efficient avoiding all that HTTP overhead.

Thrift was developed at facebook. Now I have some views about facebook due to
their primary product which is quite stupid (the users), the social network, not the
movie but the website, [facebook.com](https://facebook.com). It has almost no
credibility for common people, nothing useful, full of memes (which are
hilarious by the way if you find your way to some awesome math meme pages) and
a serious time killer. There is a [video from Veritasium](https://www.youtube.com/watch?v=LKPwKFigF8U)
about how the human mind is getting dull day by day due to these mankind's
effort of filling those spare minutes which could be used for better things.

Too much trash talk now. I actually have started to respect facebook as a tech
innovation company. Facebook has developed some pretty awesome tech like,
- [osquery](https://github.com/osquery/osquery) - Exposing the OS with an SQL
  engine on top of it.
- [Apache thrift](https://github.com/apache/thrift) - This article
- [Katran](https://github.com/facebookincubator/katran) - A library to build a
  high performance L4 loadbalancing forwarding plane using linux's XDP
  infrastructure.
- [React](https://reactjs.org) - The first choice to build a cross platform UI
  application.
- [RocksDB](https://github.com/facebook/rocksdb) - A fast multi-core version of
  levelDB.

and many more !!

## What is Thrift

Thrift lets you define an interface for how a service expects input and output.
It's like a unified way of defining a language which then can be
implemented by other languages. It's similar to defining a protocol like
[HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) which isn't
bound to any language but is a list of guidelines which are implemented by
multiple languages which allows cross language communication across processes,
applications and platforms.

Lets take a look at a dummy interface that just allows reporting a time string.
```thrift
service Timer {
    string time()
}
```
This service `Timer` returns a `string` when calling the `Timer.time` function.
Thrift works by generating boilerplate code for any supported language. We can
do so by
```sh
# MacOS install
$ brew install thrfit

# Linux
$ sudo apt install thrift
```
or visit [Project homepage](http://thrift.apache.org) for complete instructions.

Once, thrift is installed, we generate the boilerplate code using
```sh
thrift --gen py timer.thrift
```
We are using a python codegen for simplicity for the moment. We will do the same
set of activities using Rust as well.

The autogen should generate a folder `gen-py` with a bunch of python code. Let's
ignore it for now and get a server up and running to make sure it works as
expected.

```py
import sys

# Add the generated code to the path so that the boilerplate can be imported.
sys.path.append('./gen-py')
from timer import Timer

from thrift.transport import TSocket, TTransport
from thrift.protocol import TBinaryProtocol
from thrift.server import TServer

import time

# Implementation of the Service
class TimerHandler:
    def time(self):
        return str(time.time())

# Handler for the thrift calls are defined and setup
handler = TimerHandler()
proc = Timer.Processor(handler)

socket = TSocket.TServerSocket(port=9090)
transport = TTransport.TBufferedTransportFactory()
protocol = TBinaryProtocol.TBinaryProtocolFactory()
server = TServer.TSimpleServer(proc, socket, transport, protocol)

server.serve()
```
Save as `server.py`

This will basically setup the server which now accepts and replies over the thrift
protocol. A client generated from the same interface file can communicate with
this service.

Let's create a client. The boilerplate code exposes a `Client` class as well
that can be then used to communicate.
```python
import sys

# Add the generated code to the path so that the boilerplate can be imported.
sys.path.append('./gen-py')
from timer import Timer

from thrift.transport import TSocket, TTransport
from thrift.protocol import TBinaryProtocol

socket = TSocket.TSocket(port=9090)
transport = TTransport.TBufferedTransport(socket)
protocol = TBinaryProtocol.TBinaryProtocol(transport)
client = Timer.Client(protocol)

transport.open()

# Make the call to the server using the generated client.
print (client.time())
transport.close()
```
Save as `client.py`

The server can be run by just firing `python server.py` and then run `python
client.py` to run the client.

```sh
$ python server.py &
$ python client.py
1596284880.527916
```

Nice and clean !

## How does it look

Let's try and take a look at the socket dump of the request. Just set up a
wireshark dump over the loopback and it should recognize the protocol as
`THRIFT`.

![Thrift packets](/thrift_tcp_capture.png)

If we take a look at the request packet, it's just a few bits of binary message.
![Request](/thrift_request.png)

And the same is for reply, a serialized version of the response is sent as a TCP
package.
![Response](/thrift_reply.png)

Describing the protocol would be counter productive and out of scope of this
article (because I don't know how to) as it's a full blown binary
protocol that supports multiple encodings, from pure binary protocol to higher
level JSON protocol and even a space optimized zlib transport.

## Let's do the client in Rust

Now that we have a client and server, let's try to make the client in rust (
deal with only one component at a time).

We generate rust code using the `thrift` CLI.
```sh
$ cargo init thrust
$ cd thrust/src
$ thrift --gen rs /path/to/timer.thrift
```

This will generate a file called `timer.rs` with all the boilerplate code and is
simpler than the python code.

Let's write the client in Rust. We can use the Python code for inspiration or
just use the sample code in the [`thrift`](https://docs.rs/thrift) crate's example.

```rust
extern crate thrift;
mod timer;

use timer::*;
use thrift::protocol::{TBinaryInputProtocol, TBinaryOutputProtocol}
use thrift::transport::TTcpChannel;

fn main() {
    let mut channel = TTcpChannel::new();
    channel.connect("localhost:9090").unwrap();

    let (readable, writeable) = channel.split().unwrap();
    let in_stream = TBinaryInputProtocol::new(readable, true);
    let out_stream = TBinaryOutputProtocol::new(writeable_true);

    let mut client = TimerSyncClient::new(in_stream, out_stream);
    println!("{:?}", client.time());
}
```
If you do a `cargo run`, It should return a similar output as the python client.
Pretty simple, didn't require a lot of effort, example's got you.

## Let's get into the unix socket.

[Unix sockets](http://beej.us/guide/bgipc/html/multi/unixsock.html)
are Full duplex named sockets that are usually meant for inter-process
communication. It's relatively faster than using the loopback mechanism because
it avoids the data going over the whole TCP Stack that includes the routing
mechanism as well since it's going over an interface. Due to the same reason, it
would be more efficient if we route the thrift traffic over a unix domain socket
instead of localhost.

But, the `thrift` crate doesn't show any example or API to be able to connect
via Unix Sockets. It only has a
[`thrift::transport::TTcpChannel`](https://docs.rs/thrift/0.13.0/thrift/transport/struct.TTcpChannel.html)
which can't be used with the Unix Sockets as it doesn't use the TCP Stack at
all.

Now, let's see what can be done. Let's explore the `TimerSyncClient`'s
signature from the generated `timer.rs` file.

```rust
pub struct TimerSyncClient<IP, OP> where IP: TInputProtocol, OP: TOutputProtocol {
  _i_prot: IP,
  _o_prot: OP,
  _sequence_number: i32,
}

impl <IP, OP> TimerSyncClient<IP, OP> where IP: TInputProtocol, OP: TOutputProtocol {
  pub fn new(input_protocol: IP, output_protocol: OP) -> TimerSyncClient<IP, OP> {
    TimerSyncClient { _i_prot: input_protocol, _o_prot: output_protocol, _sequence_number: 0 }
  }
}
```

So, the client needs `thrift::protocol::{TInputProtocol, TOuptutProtocol}`
for initialization.

Upon inspection of those traits, not much insight is gained.

Let's try looking into the `readable` and `writeable` streams that the TCP
example has. `thrift::tranport::TIoChannel` has a `split` method that returns a
```rust
fn split(self) -> Result<(ReadHalf<Self>, WriteHalf<Self>)>
```
Ok, so looks like it needs individual streams to be able to read and write.
Let's inspect these structs.

An instance of the `ReadHalf` struct can be created as
```rust
pub fn new(handle: C) -> ReadHalf<C>
where
    C: Read,
```

So, we just need a struct which implements a `Read` and we should be good
to create a `ReadHalf` implementation, and same for `WriteHalf`, we just need a
`Write` implementation.

Let's roll back to our unix sockets from the standard library.
[`std::os::unix::net::UnixStream`](https://doc.rust-lang.org/std/os/unix/net/struct.UnixStream.html)

From the docs we see that it has the following traits implemented.
- [`Read`](https://doc.rust-lang.org/std/os/unix/net/struct.UnixStream.html#impl-Read)
- [`Write`](https://doc.rust-lang.org/std/os/unix/net/struct.UnixStream.html#impl-Write)


Which means, [`UnixStream`](https://doc.rust-lang.org/std/os/unix/net/struct.UnixStream.html)
should suffice. Let's get to it then.

```rust
extern crate thrift;
mod timer;

use timer::*;
use thrift::protocol::{TBinaryInputProtocol,TBinaryOutputProtocol};
use std::os::unix::net::UnixStream;
use std::io::prelude::*;

fn main() {
    let socket_tx = UnixStream::connect("/tmp/timer.sock").unwrap();
    let socket_rx = socket_tx.try_clone().unwrap();

    let in_proto = TBinaryInputProtocol::new(socket_tx, true);
    let out_proto = TBinaryOutputProtocol::new(socket_rx, true);
    let mut client = TimerSyncClient::new(in_proto, out_proto);

    println!("{:?}", client.time());
}
```

Change the python server to listen on a unix socket instead of a TCP socket,
i.e.
```python
socket = TSocket.TServerSocket(unix_socket='/tmp/timer.sock')
```
And, then a `cargo run` and **Success!!**. We get the time!. Too much trouble to
just get a time string, but it's worth it B).

> Note that we require to clone the socket. This is because
> [`TBinaryInputProtocol::new`](https://docs.rs/thrift/0.13.0/thrift/protocol/struct.TBinaryInputProtocol.html#method.new)
> takes the ownership of the `transport: T`. The `UnixStream` is full duplex,
> which means the same object will do a read and write, so we need to get
> individual copies of the socket for BinaryIn and BinaryOut protocols.

And that was it. At the moment, I don't have a good enough understanding of the
rust thrift crate to be able to figure out how to get it to listen on a
Unix Socket since it only has a listen method on a [`TSocket`](https://docs.rs/thrift/0.13.0/thrift/server/struct.TServer.html#method.listen)
struct that accepts a _host:port_ combo.
```rust
pub fn listen(&mut self, listen_address: &str) -> Result<()>
```

## Thrift Server over TCP
As a consolation, just to make sure, the TCP server works, here is the
implementation. Most of it can be taken from either the python code or
[examples](https://docs.rs/thrift/0.13.0/thrift/server/struct.TServer.html#examples)
in the [`thrift`](https://docs.rs/thrift) crate.
```rust
impl TimerSyncHandler for TimerSyncHandlerImpl {
    fn handle_time(&self) -> thrift::Result<String> {
        return Ok(format!(
            "{:?}",
            SystemTime::now().duration_since(SystemTime::UNIX_EPOCH)
        ));
    }
}

fn server() {
    let processor = TimerSyncProcessor::new(TimerSyncHandlerImpl {});

    let i_tr_fact: Box<TReadTransportFactory> = Box::new(TBufferedReadTransportFactory::new());
    let i_pr_fact: Box<TInputProtocolFactory> = Box::new(TBinaryInputProtocolFactory::new());
    let o_tr_fact: Box<TWriteTransportFactory> = Box::new(TBufferedWriteTransportFactory::new());
    let o_pr_fact: Box<TOutputProtocolFactory> = Box::new(TBinaryOutputProtocolFactory::new());

    let mut server = TServer::new(i_tr_fact, i_pr_fact, o_tr_fact, o_pr_fact, processor, 10);
    server.listen("localhost:9090").unwrap();
}

fn main() {
    let t = thread::spawn(|| { server(); });

    /* client code */
    // The client will connect to the server, print the output and then
    // start listening for other connections.

    t.join().unwrap();
}
```
---
For now, the client implementation using unix sockets is sufficient as the aim
for this is to be able to communicate to `/var/osquery/osquery.em` socket which
I hope is a unix socket.

---
Discussion thread [here](https://www.reddit.com/r/rust/comments/i70l4b/apache_thrift_over_unix_sockets_in_rust/)
