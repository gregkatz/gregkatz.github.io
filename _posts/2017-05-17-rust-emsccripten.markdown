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

```
curl -O https://s3.amazonaws.com/mozilla-games/emscripten/releases/emsdk-portable.tar.gz
tar -xzf emsdk-portable.tar.gz
source emsdk_portable/emsdk_env.sh
emsdk update
emsdk install sdk-incoming-64bit
emsdk activate sdk-incoming-64bit
```

[users-guide]: https://users.rust-lang.org/t/compiling-to-the-web-with-rust-and-emscripten/7627
[game]: https://www.github.com/gregkatz/cvsolitaire
