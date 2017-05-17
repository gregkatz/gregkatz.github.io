---
layout: post
title:  "Deploying graphical applications to the web with Rust and emscripten"
date:   2017-05-17 14:37:10 -0400
categories: Rust emscripten SDL webassembly Asm.js
---
After recently writing a simple cross-platform [solitaire game][game] inspired by Shenzhen IO, I thought: "What next?" I remembered a post I had read last year by Brian Anderson titled "[Compiling to the web with Rust and emscripten][users-guide]." In this post, I'll describe how I applied that guide to get my solitaire game (mostly) running in the browser.

(My apologies to Windows developers, but this post will focus on Linux and Mac.)

## Initial setup
I'm assuming you have Rust installed through rustup.rs. Brian's guide starts by telling you to switch to nightly, but this is no longer required since 1.14. So enjoy the stable life.

First, we'll add new targets to rustup, so we can compile to them later. We'll be adding the asmjs and wasm32 targets. Wasm is newer and faster, but relies on your browser's webassembly support. As of writing, Chrome and Firefox support webassembly out of the box. Asmjs should have more broad compatibility, at the expense of speed.

```
rustup target add asmjs-unknown-emscripten
rustup target add wasm32-unknown-emscripten
```

Now we'll need to install the emscripten SDK. This hasn't changed since Brian's guide, so we'll copy him exactly. Go to your miscleanious projects directory and run:
(This took nearly an hour for me.)
```
curl -O https://s3.amazonaws.com/mozilla-games/emscripten/releases/emsdk-portable.tar.gz
tar -xzf emsdk-portable.tar.gz
source emsdk_portable/emsdk_env.sh
emsdk update
emsdk install sdk-incoming-64bit
emsdk activate sdk-incoming-64bit
```

I had to run ```source emsdk_portable/emsdk_env.sh``` one more time after too. Check that ```emcc``` is in your path to make sure it worked.

Again, following Brian's guide, we'll do a quick test:
```
echo 'fn main() { println!("Hello, Emscripten!"); }' > hello.rs
rustc --target=asmjs-unknown-emscripten hello.rs
node hello.js
```

If that worked, we're good to move on.

##Building something bigger
First, lets clone the solitaire game. The game was written to have minimal dependencies for Redox compatability, which also makes it a good fit for emscripten. Go to your projects folder and run:
```
git clone https://www.github.com/gregkatz/cvsolitaire
```

Now, make sure you can build it natively before we start cross compiling:

```
cargo build
```

If that didn't work, it may be because the game does have an SDL2 dependency so you'll need to install it first. On a Mac using Homebrew that would be ```brew install sdl2```. On Ubuntu it's probably ```sudo apt-get install libsdl2-dev```.

Once you've got it building on native, lets try building to one of the emscripten targets. After all that's why you're here. You may remember that Brian was using rustc in his guide. We can't do that, because it would be too burdensome to build all of the game's dependencies with rustc and manually link. Forutunately, cargo accepts the same cross-compiling arguments:

```
cargo build --release --target=asmjs-unknown-emscripten
```
So it failed. But it failed on linking! I was astonished that it got even that far. I also tried building some other projects of mine and they certainly didn't get that far.

[users-guide]: https://users.rust-lang.org/t/compiling-to-the-web-with-rust-and-emscripten/7627
[game]: https://www.github.com/gregkatz/cvsolitaire
