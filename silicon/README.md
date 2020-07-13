# Setup

- Download and install Xcode 12 beta, then set it as your default

    ```
    xcode-select -s /Applications/Xcode-beta.app/Contents/Developer/
    ```

- Clone [this fork of rust][fork], checkout the "silicon" branch.

[fork]: https://github.com/shepmaster/rust

# Build a compiler capable of cross-compiling to the DTK

The current Rust compiler doesn't know how to target the DTK. We need
to build an enhanced version that can.

This section can be removed when an appropriate target specification
is added to the standard Rust bootstrapping compiler.

Theoretically, these steps can be done on the DTK itself, but that
hasn't been tested yet.

1. `cd silicon/bootstrap`

1. Build our bootstrapping compiler

    ```
    SDKROOT=$(xcrun -sdk macosx11.0 --show-sdk-path) \
    MACOSX_DEPLOYMENT_TARGET=$(xcrun -sdk macosx11.0 --show-sdk-platform-version) \
    ../../x.py build -i --stage 1 \
    src/libstd
    ```

  N.B. these environment variables *probably* aren't required, but
  they are for consistency with the next step.

# Cross-compile the compiler for the DTK

Now that we have a compiler that knows about the DTK target, we can
build a compiler native to the DTK.

1. Modify the absolute paths in `silicon/cross/config.toml` in the
   `cargo`, `rustc`, and `rustfmt` keys to point to your checkout.

1. `cd silicon/cross`

1. Cross-compile the compiler

    ```
    SDKROOT=$(xcrun -sdk macosx11.0 --show-sdk-path) \
    MACOSX_DEPLOYMENT_TARGET=$(xcrun -sdk macosx11.0 --show-sdk-platform-version) \
    RUST_DESTDIR=/tmp/crossed \
    ../../x.py install -i --stage 1 --host aarch64-apple-darwin --target aarch64-apple-darwin,x86_64-apple-darwin \
    src/librustc src/libstd cargo
    ```

# Copy the cross-compiler to the DTK

`rsync` is a good choice, as you'll probably need to iterate a few times!

```
rsync -avz /tmp/crossed/ dtk:crossed/
```

# Build a native binary

On the DTK, you can now use native Rust:

```
% echo 'fn main() { println!("Hello, DTK!") }' > hello.rs

% ~/crossed/usr/local/bin/rustc hello.rs

% ./hello
Hello, DTK!

% file hello
hello: Mach-O 64-bit executable arm64
```

# Installing rustup

Rustup doesn't know that `x86_64` binaries can run on `aarch64`, so we
have to hack the install script so it thinks that the machine is
`x86_64`.

1. Download the installer script

    ```
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs > /tmp/rustup.rs
    ```

1. Apply this patch to force it to think we are x86_64:

    ```diff
    --- /tmp/rustup.rs  2020-07-05 07:49:10.000000000 -0700
    +++ /tmp/rustup-hacked.rs   2020-07-04 17:27:08.000000000 -0700
    @@ -188,6 +188,8 @@
             fi
         fi

    +    _cputype=x86_64
    +
         case "$_ostype" in

             Android)
    ```

1. Install rustup

    ```
    bash /tmp/rustup.sh --default-toolchain none
    ```

1. Create a toolchain and make it the default:

    ```
    rustup toolchain link native ~/crossed/usr/local/
    rustup default native
    ```

# Build the compiler on the DTK

Re-run the setup steps on the DTK, if you haven't already.

1. `cd silicon/pure-native`

1. Get an x86_64 version of Cargo to use during bootstrapping

    ```
    curl 'https://static.rust-lang.org/dist/2020-06-16/cargo-beta-x86_64-apple-darwin.tar.xz' -o cargo.tar.xz
    tar -xf cargo.tar.xz
    ```

1. Build and install the compiler

    ```
    RUST_DESTDIR=~/native-build \
    ../../x.py install -i --stage 1 --host aarch64-apple-darwin --target aarch64-apple-darwin,x86_64-apple-darwin \
    src/librustc src/libstd cargo
    ```

# Miscellaneous notes

- `SDKROOT` - used to indicate to `rustc` what SDK whould be
  used. Point this at the `MacOSX11` target inside your `Xcode.app` /
  `Xcode-beta.app`

- `MACOSX_DEPLOYMENT_TARGET` - used to indicate to `rustc` what
  version of macOS to target.

- We build LLVM *3 times* and use incremental builds for faster
  iteration times. This can easily take up **30 GiB** of
  space. Building can take several hours on an 2.9 GHz 6-Core Intel
  Core i9.

# Troubleshooting

- Failure while running `x.py`? Add `-vvvvvvv`.

- "archive member 'something.o' with length xxxx is not mach-o or llvm bitcode file 'some-tmp-path/libbacktrace_sys-e02ed1fd10e8b80d.rlib' for architecture arm64"?
  - Find the rlib: `find build -name 'libbacktrace_sys*'`
  - Remove it: `rm build/aarch64-apple-darwin/stage0-std/aarch64-apple-darwin/release/deps/libbacktrace_sys-e02ed1fd10e8b80d.r{meta,lib}`
  - Look at the objects it created in `build/aarch64-apple-darwin/stage0-std/aarch64-apple-darwin/release/build/backtrace-sys-fec595dd32504c90/`
  - Delete those too: `rm -r build/aarch64-apple-darwin/stage0-std/aarch64-apple-darwin/release/build/backtrace-sys-fec595dd32504c90`
  - Rebuild with verbose
