# WASI tutorial
We'll split the tutorial into two parts: in the first part we'll walk through
compiling C and Rust programs to WASI, and in the second part how to execute
the compiled WebAssembly module using `wasmtime` runtime.

- [WASI tutorial](#wasi-tutorial)
  - [Compiling to WASI](#compiling-to-wasi)
    - [From C](#from-c)
    - [From Rust](#from-rust)
  - [Executing in `wasmtime` runtime](#executing-in-wasmtime-runtime)

## Compiling to WASI
### From C
Let's start with a simple C program which performs a file copy, which will
show to compile and run programs, as well as perform simple sandbox
configuration. The C code here uses standard POSIX APIs, and doesn't have
any knowledge of WASI, WebAssembly, or sandboxing.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

int
main(int argc, char **argv) {
    int n, m;
    char buf[BUFSIZ];

    if (argc != 3) {
        fprintf(stderr, "usage: %s <from> <to>\n", argv[0]);
        exit(1);
    }

    int in = open(argv[1], O_RDONLY);
    if (in < 0) {
        fprintf(stderr, "error opening input %s: %s\n", argv[1], strerror(errno));
        exit(1);
    }

    int out = open(argv[2], O_WRONLY | O_CREAT, 0660);
    if (out < 0) {
        fprintf(stderr, "error opening output %s: %s\n", argv[2], strerror(errno));
        exit(1);
    }

    while ((n = read(in, buf, BUFSIZ)) > 0) {
        while (n > 0) {
            m = write(out, buf, n);
            if (m < 0) {
                fprintf(stderr, "write error: %s\n", strerror(errno));
                exit(1);
            }
            n -= m;
        }
    }

    if (n < 0) {
        fprintf(stderr, "read error: %s\n", strerror(errno));
        exit(1);
    }

    return EXIT_SUCCESS;
}
```

We'll put this source in a file called `demo.c`.

The [wasi-sdk](https://github.com/CraneStation/wasi-sdk/releases) provides a clang
which is configured to target WASI and use the WASI sysroot by default, so we can
compile our program like so:

```
$ clang demo.c -o demo.wasm
```

A few things to note here. First, this is just regular clang, configured to use
a WebAssembly target and sysroot. Second, the output name specified with the "-o"
flag can be anything you want, and *does not* need to contain the `.wasm` extension.
In fact, the output of clang here is a standard WebAssembly module:

```
$ file demo.wasm
demo.wasm: WebAssembly (wasm) binary module version 0x1 (MVP)
```


### From Rust
The same effect can be achieved with Rust. Firstly, go ahead and create a new
binary crate:

```
$ cargo new --bin demo
```

You can also clone the Rust code with the crate preset for you from
[here](https://github.com/kubkon/rust-wasi-tutorial).

Now, let's port the C program defined in [From C](#from-c) section to Rust:

```rust
use std::env;
use std::fs;
use std::io::{Read, Write};

fn process(input_fname: &str, output_fname: &str) -> Result<(), String> {
    let mut input_file =
        fs::File::open(input_fname).map_err(|err| format!("error opening input: {}", err))?;
    let mut contents = Vec::new();
    input_file
        .read_to_end(&mut contents)
        .map_err(|err| format!("read error: {}", err))?;

    let mut output_file = fs::File::create(output_fname)
        .map_err(|err| format!("error opening output '{}': {}", output_fname, err))?;
    output_file
        .write_all(&contents)
        .map_err(|err| format!("write error: {}", err))
}

fn main() {
    let args: Vec<String> = env::args().collect();
    let program = args[0].clone();

    if args.len() < 3 {
        eprintln!("{} <input_file> <output_file>", program);
        return;
    }

    if let Err(err) = process(&args[1], &args[2]) {
        eprintln!("{}", err)
    }
}
```

Let's put this source in the main file of our crate `src/main.rs`.

In order to build it, we first need to install a WASI-enabled Rust toolchain:

```
$ rustup target add wasm32-unknown-wasi --toolchain nightly
$ cargo +nightly build --target wasm32-unknown-wasi
```

We should now have the WebAssembly module created in `target/wasm32-unknown-wasi/debug`:

```
$ file target/wasm32-unknown-wasi/debug/demo.wasm
demo.wasm: WebAssembly (wasm) binary module version 0x1 (MVP)
```

## Executing in `wasmtime` runtime
The resultant WebAssembly module `demo.wasm` compiled either from C or Rust is simply
a single file containing a self-contained wasm module, that doesn't require
any supporting JS code.

We can execute it with `wasmtime` directly, like so:

```
$ wasmtime demo.wasm
usage: demo.wasm <from> <to>
```

Ok, this program needs some command-line arguments. So let's give it some:

```
$ echo hello world > test.txt
$ wasmtime demo.wasm test.txt /tmp/somewhere.txt
error opening input test.txt: Capabilities insufficient
```

Aha, now we're seeing the sandboxing in action. This program is attempting to
access a file by the name of `test.txt`, however it hasn't been given the
capability to do so.

So let's give it capabilities to access files in the requisite directories:

```
$ wasmtime --dir=. --dir=/tmp demo.wasm test.txt /tmp/somewhere.txt
$ cat /tmp/somewhere.txt
hello world
```

Now our program runs as expected!

What's going on under the covers? The `--dir=` option instructs `wasmtime`
to *preopen* a directory, and make it available to the program as a capability
which can be used to open files inside that directory. Now when the program
calls the C/Rust `open` function, passing it either an absolute or relative path,
the WASI libc transparently translates that path into a path that's relative to
one of the given preopened directories, if possible (using a technique based
on [libpreopen](https://github.com/musec/libpreopen)). This way, we can have a
simple capability-oriented model at the system call level, while portable
application code doesn't have to do anything special.

As a brief aside, note that we used the path `.` above to grant the program
access to the current directory. This is needed because the mapping from
paths to associated capabilities is performed by libc, so it's part of the
WebAssembly program, and we don't expose the actual current working
directory to the WebAssembly program. So providing a full path doesn't work:

```
$ wasmtime --dir=$PWD --dir=/tmp demo.wasm test.txt /tmp/somewhere.txt
$ cat /tmp/somewhere.txt
error opening input test.txt: Capabilities insufficient
```

So, we always have to use `.` to refer to the current directory.

Speaking of `.`, what about `..`? Does that give programs a way to break
out of the sandbox? Let's see:

```
$ wasmtime --dir=. --dir=/tmp demo.wasm test.txt /tmp/../etc/passwd
error opening output /tmp/../etc/passwd: Capabilities insufficient
```

The sandbox says no. And note that this is the capabilities system saying no
here ("Capabilities insufficient"), rather than Unix access controls
("Permission denied"). Even if the user running `wasmtime` had write access to
`/etc/passwd`, WASI programs don't have the capability to access files outside
of the directories they've been granted. This is true when resolving symbolic
links as well.

`wasmtime` also has the ability to remap directories, with the `--mapdir`
command-line option:

```
$ wasmtime --dir=. --mapdir=/tmp:/var/tmp demo.wasm test.txt /tmp/somewhere.txt
$ cat /var/tmp/somewhere.txt
hello world
```

This maps the name `/tmp` within the WebAssembly program to `/var/tmp` in the
host filesystem. So the WebAssembly program itself never sees the `/var/tmp` path,
but that's where the output file goes.

See [here](WASI-capabilities.md) for more information on the capability-based
security model.

The capability model is very powerful, and what's shown here is just the beginning.
In the future, we'll be exposing much more functionality, including finer-grained
capabilities, capabilities for network ports, and the ability for applications to
explicitly request capabilities.
