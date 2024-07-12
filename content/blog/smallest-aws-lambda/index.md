+++
title = "Smallest AWS Lambda function"
description = "How small could you make an AWS Lambda with a runtime."
date = 2024-07-11
[taxonomies]
tags = ["AWS", "Linux"]
+++

## Backstory
During my time at university I was playing around AWS Lambda functions and runtime, and a random thought accord to me,
how small can I make a Lambda function and a runtime

## Quest of the tiny
AWS provides a bunch of documentation about [Lambda execution environment](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html) and how you build a [custom runtime](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html),
of course there are many challenges and things to consider when building a production grade runtime,
but I'm just experimenting here, it's not my server that will catch on fire
![Architecture diagram of Lambda execution environment](https://docs.aws.amazon.com/images/lambda/latest/dg/images/telemetry-api-concept-diagram.png)
From what I understand the whole thing ~roughly works as follows:
1. Initialize the runtime
1. We poll Lambda Runtime API for the next invocation
1. The runtime pass the request data with an ID to the request handler for processing
1. The request handler sends the response to the Runtime API using the provided ID
1. Loop again from step 2

Seems doable, let's give it a try and see how small we can make it


## Attempt #1: Go Lang
Since I will be required to do network requests, Go lang seems like a good fit for it's excellent standard library,
and Go lang being a compiled language would help with the binary size
```go
// ...
for {
    // get the next invocation
    url := fmt.Sprintf("http://%s/2018-06-01/runtime/invocation/next", RUNTIME_API)
    resp, _ := http.Get(url)
    req_id := resp.Header.Get("Lambda-Runtime-Aws-Request-Id")

    // respond to the request
    url = fmt.Sprintf("http://%s/2018-06-01/runtime/invocation/%s/response", RUNTIME_API, req_id)
    http.Post(
        url,
        "",
        strings.NewReader("Hello, World!"))

    resp.Body.Close()
}
```
I really like how easy to get things done in Go, it's was very enjoyable to implement and incredibly pleasant to use

Let's try to compile and we get this
```bash
$ go build
$ ls -lh bootstrap | awk '{print $5, $9}'
6.9M bootstrap
```

seems odd for such a simple binary,
as it turns out Go includes debug symbols by default,
so let's try to remove them
```bash
$ strip bootstrap
$ ls -lh bootstrap | awk '{print $5, $9}'
4.7M bootstrap
```

much better!

## Attempt #2 Rust Lang
While Go got us through the first implementation, it's gonna be really hard to get rid of the entire runtime to achieve a smaller binary,
so I turn my eyes to Rust, the promise of having _almost_ no runtime seems to fit my goal for the smallest binary
```rust
// ...
loop {
    // gets the next invocation
    let url = format!("http://{runtime_api}/2018-06-01/runtime/invocation/next");
    let request = ureq::get(&url).call()?;
    let req_id = request
        .header("Lambda-Runtime-Aws-Request-Id")
        .ok_or("ERROR: Bad request")?;

    // responds to the invocation
    let url = format!("http://{runtime_api}/2018-06-01/runtime/invocation/{req_id}/response");
    ureq::post(&url).send_string("Hello, World!")?;
}
```

so let's try to compile and see what we get
```sh
$ cargo build --release
$ du -h target/release/bootstrap
$ ls -lh target/release/bootstrap | awk '{print $5, $9}'
2.6M target/release/bootstrap
```

not too bad, let's try to play with the compiler nobs
quick searching in the web gave me these
```toml
# Cargo.toml
# ...
[profile.release]
strip = true
opt-level = "z"
lto = true
panic = "abort"
```

compile again and we save an entire whopping _megabyte_...
```sh
$ cargo build --release
$ ls -lh bootstrap | awk '{print $5, $9}'
1.6M bootstrap
```

this was so smooth, let's try to run it and... we get a bunch of errors _ofc_
```
/var/task/bootstrap: /lib64/libc.so.6: version `GLIBC_2.28' not found (required by /var/task/bootstrap)
/var/task/bootstrap: /lib64/libc.so.6: version `GLIBC_2.33' not found (required by /var/task/bootstrap)
/var/task/bootstrap: /lib64/libc.so.6: version `GLIBC_2.32' not found (required by /var/task/bootstrap)
/var/task/bootstrap: /lib64/libc.so.6: version `GLIBC_2.34' not found (required by /var/task/bootstrap)
```

apparently Rust doesn't like the version of `GLIBC` provided by Amazon Linux 2,
so I guess I'll just have to statically link with `musl` maybe, let me try...
```sh
$ cargo target add x86_64-unknown-linux-musl
$ cargo build --release --target x86_64-unknown-linux-musl
$ ls -lh bootstrap | awk '{print $5, $9}'
1.7M bootstrap
```
Hold on!! it adds an entire 0.1Mb, I can't have that, not on my watch, there must be another way

I kept looking around and found that using Amazon Linux 2023 instead of Amazon Linux 2 solved my problems with `GLIBC`,
but at this point I'm too tired so you know what Rust, maybe we should see other people

## Attempt #3 Assembly
Of course the first thing we have the implement for is the _entire_ 114 pages of the [HTTP Spec...](https://www.rfc-editor.org/rfc/rfc2616.pdf)

Well I guess have to pack my things and go Home

_or..._ I could a Frankencode monster that somehow gets the job done

See I don't have to implement the whole `HTTP` Protocol to get my experiment working,
if you think about it, a protocol is just a way of arranging bytes to transmit data between computers,
so for my need I only need to arrange some bytes to handcraft a valid `HTTP` message and I should be in a good shape

Since we don't care about `HTTP` spec much, here's my evil plan:
1. Open a `TCP` socket on the lambda runtime api endpoint
1. Shove some bytes of `HTTP` to get the next invocation
1. Read the data from the socket into the stack
1. Copy the bytes of request ID into the response
1. Shove the response back into the socket

And after many hours of researching [x86 instruction reference](https://www.felixcloutier.com/x86/), [x86_64 calling convention](https://en.wikipedia.org/wiki/X86_calling_conventions), [Linux Syscalls](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md), and [Fasm Documentation](https://flatassembler.net/docs.php?article=manual) and with countless hours of blind debugging, I finally managed to get it working

So here I present to you my masterpiece (or monstrosity depending on who you ask...)

```asm
; ...
main:
    ; opens a socket
    socket AF_INET, SOCK_STREAM, 0
    mov rbx, rax ; store socket_fd in rbx

    ; connects to the socket
    connect rbx, servaddr.sin_family, sizeof_servaddr

    ; allocates 1Mb of stack memory for reading the request
    sub rsp, 1048576
l:
    ; gets the next invocation
    write rbx, get_request, get_request_len

    ; reads the response from the socket
    read rbx, rsp, 1048576

    ; memcpy request_id into the response
    mov rdi, post_request + 36
    mov rsi, rsp
    add rsi, 80 ; request_id offset in the response
    mov rcx, 36 ; request_id_len
    rep movsb

    ; responds to the request
    write rbx, post_request, post_request_len
    read rbx, rsp, 1048576
    jmp l
```

And it freaking works!!
```sh
$ aws lambda invoke --function-name assembly-runtime output.txt
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
$ cat output.txt
Hello, World!
```

And now for the moment of truth, let's see how small is it
```sh
$ fasm bootstrap.asm
$ ls -lh bootstrap | awk '{print $5, $9}'
592 bootstrap
```
just a mere 592 _bytes_, not even a kilobyte, it's so small that's it's an order of magnitude smaller than the page you're currently reading


funny enough the binary is smaller than the source code that made it
```sh
$ ls -lh bootstrap.asm | awk '{print $5, $9}'
1.9K bootstrap.asm
```
> ![bootstrap-asm-qr.base64](bootstrap-asm-qr.svg#end)
<br />
<br />
<br />
> It's so small that the entire binary (base64 encoded) could fit in a QR code!!
<br />
<br />
<br />
<br />

I think there are still ways to shave of some bytes of the binary but at I've lost so much hair already debugging this,
so I'm gonna leave that as an "exercise for the reader", the entirety for the code is on this [repo](https://github.com/pinjeff/smallest-aws-lambda)

## Goal Achieved?
Now that we're at the end, do this have any real world use?
probably no, but that wasn't my goal...

I've learned a lot through this I had to know the machinery behind the Lambda Runtime,
then how build a runtime for in Go which had me dealing with striping binaries,
then moving to Rust and dealing with compiler optimizations and cross-compiling,
and then to Assembly and dealing with the Linux kernel and functions calling convention
and add to that a better understanding of HTTP, the whole journey had me going over lots of stuff

So in the end I really enjoyed diving deep into this rabbit hole,
it was really enjoyable the whole way through ~except debugging assembly, that's still hell ;)~

