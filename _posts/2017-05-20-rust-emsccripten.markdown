---
layout: default
title:  "Building graphical applications to js in stable Rust"
date:   2017-05-20 7:00:00 -0400
categories: Rust emscripten SDL webassembly Asm.js
permalink: /2017-05-20-rust-emscripten.html
---
## Building graphical applications to JS in stable Rust

![CVSolitaire running in firefox]({{ site.url }}/assets/CVSolitaire_wasm.png)

May 20, 2017

If you would rather [just try the demo][demo] or check out the [repo][repo], be warned: ~7mb of JS and [BUGS](#bugs).

After recently writing a simple cross-platform [solitaire game][game] inspired by Shenzhen IO, I thought: "What next?" I remembered a post I had read last year by brson titled "[Compiling to the web with Rust and emscripten][users-guide]." This post describes how to apply that guide to get my solitaire game (mostly) running in the browser.

(Apologies to Windows users, but this post focuses on Linux and Mac.)

## Initial setup
This post assumes that you have Rust installed through rustup.rs. Brson's guide starts by telling you to switch to nightly, but this is no longer required since Rust 1.14. Enjoy the stable life.

First, add the asmjs and wasm32 targets via rustup:

{% highlight bash %}
rustup target add asmjs-unknown-emscripten
rustup target add wasm32-unknown-emscripten
{% endhighlight %}

Wasm is newer and faster, but relies on your browser's webassembly support. As of writing, Chrome and Firefox support webassembly out of the box. Asmjs should have broader compatibility, at the expense of speed.

Now, install the emscripten SDK. This has not changed since brson's guide, so copy him exactly.
Go to your miscellaneous projects directory and run:
{% highlight bash %}
curl -O https://s3.amazonaws.com/mozilla-games/emscripten/releases/emsdk-portable.tar.gz
tar -xzf emsdk-portable.tar.gz
source emsdk-portable/emsdk_env.sh
emsdk update
emsdk install sdk-incoming-64bit
emsdk activate sdk-incoming-64bit
{% endhighlight %}

(This took nearly an hour for me.)
You may need to run ```source emsdk-portable/emsdk_env.sh``` one extra time. Check that ```emcc``` is in your path to make sure the SDK is installed.

Again, following brson's guide, do a quick test:
{% highlight bash %}
echo 'fn main() { println!("Hello, Emscripten!"); }' > hello.rs
rustc --target=asmjs-unknown-emscripten hello.rs
node hello.js
{% endhighlight %}

If that works, continue on.

## Building something bigger
First, clone the solitaire game. I wrote the game to have minimal dependencies for [Redox][redox] compatibility, which also makes it a good fit for emscripten. It also embeds its sprite assets in the binary which allows you to avoid simulating a filesystem with emscripten.
Go to your projects folder and run:
{% highlight bash %}
git clone https://www.github.com/gregkatz/cvsolitaire
{% endhighlight %}

Now, make sure you can build natively before you start cross compiling:

{% highlight bash %}
cd cvsolitaire
cargo build
{% endhighlight %}

If it fails, you may be missing the SDL2 dependency. To install SDL2 on a Mac using Homebrew run: ```brew install sdl2```. On Ubuntu, run: ```sudo apt-get install libsdl2-dev```.

Once you have built successfully on native, try building to one of the emscripten targets. You may remember that brson used rustc directly in his guide. It would be too burdensome to build all of the game's dependencies with rustc and link them manually. Luckily, cargo accepts the same cross-compilation arguments:

{% highlight bash %}
cargo build --target=asmjs-unknown-emscripten
{% endhighlight %}
This fails. But it fails on linking! I was astonished that it compiles successfully. I also tried building some of my other projects, but no luck.

## Getting SDL for emscripten
The build produces a lot of output, but the important parts are:
```
error: linking with `emcc` failed: exit code: 1
```

```
error: unresolved symbol: SDL_WaitEvent
error: unresolved symbol: SDL_GetWindowTitle
error: unresolved symbol: SDL_GetWindowPosition
error: unresolved symbol: SDL_CreateWindow
error: unresolved symbol: SDL_CreateRenderer
error: unresolved symbol: SDL_RenderPresent
error: unresolved symbol: SDL_GetWindowSurface
```

Basically, the issue is a missing dependency on SDL2. As it turns out, there are a select few libraries that ported over to [emscripten-ports][ports] and SDL2 is one of them.

As described in [this][c-guide] guide, using SDL2 on emscripten is as easy as passing ```"-s" "USE_SDL=2"``` to the linker.

As best as I can tell there is no way to pass most linker arguments through cargo. Instead, hack around the limitation by specifying a custom linker that is actually a shell script wrapping the real linker. (If anyone knows a better way, please tell me and I will update the post.)

In your project folder, make a new file called ```emcc_sdl``` and add:
{% highlight bash %}
emcc "-s" "USE_SDL=2" $@
{% endhighlight %}

Run ```chmod +x emcc_sdl``` to make it executable. This tiny script calls the linker, adds our SDL2 arguments and passes along the other arguments it was called with.

Now, tell cargo to use the new "linker" instead of the real one. Create ```.cargo/config``` and insert the following lines:
{% highlight toml %}
[target.wasm32-unknown-emscripten]
linker = "/your/project/dir/cvsolitaire/emcc_sdl"

[target.asmjs-unknown-emscripten]
linker = "/your/project/dir/cvsolitaire/emcc_sdl"
{% endhighlight %}
Replace ```/your/project/dir/``` with the actual location of your project. This tells cargo to use our custom linker when compiling. Now build again, and it should succeed!

But what have you accomplished? Check out ```./target/asmjs-unknown-emscripten/debug/``` and you should find ```cvsolitaire.js``` waiting for you.

## Getting it running
Earlier, you ran the JS in node and let it print to console. That approach no longer works because the JS needs to draw the graphics to an HTML canvas. Fortunately, emscripten can generate the HTML automatically. Edit ```emcc_sdl``` to add a new output flag and also to tell emcc to optimize the build. (Note: this causes the compiler to output to the current directory.)

Change ```emcc_sdl``` to read:
{% highlight bash %}
emcc "-s" "USE_SDL=2" "-o" "cvsolitaire.html" "-02" $@
{% endhighlight %}

Build again and you have HTML to go along with the JS.
{% highlight bash %}
cargo clean
cargo build --target=asmjs-unknown-emscripten
{% endhighlight %}

But don't open that HTML!
## Finishing touches
Opening the HTML causes the browser to lock up until the unresponsive script dialog can interrupt. The problem, as [described here][em-sdl], is that SDL's event loop monopolizes the browser's engine, and prevents it from doing anything else. To fix the issue, allow emscripten to throttle the event loop so the browser can do other things.

First, edit ```src/main.rs``` to remove the very last line that reads:
{% highlight rust %}
window.exec();
{% endhighlight %}
Instead, place a single iteration of the event loop in a closure, which emscripten calls when the browser has resources available:
{% highlight rust %}
set_main_loop_callback(||{
    window.drain_events();
    window.draw_if_needed();
    window.drain_orbital_events();
});
{% endhighlight %}

Next, borrow some [code][triangle-repo] developed by [badboy_][jan-home] for his talk "[Compiling Rust to your Browser][talk]" which wraps the closure in a function, and passes the function as a callback to emscripten. Put the following after the main function:

{% highlight rust %}
use std::os::raw::{c_void, c_int};
use std::ptr::null_mut;

#[allow(non_camel_case_types)]
type em_callback_func = unsafe extern fn();
extern {
    fn emscripten_set_main_loop(func: em_callback_func,
                                fps: c_int,
                                simulate_infinite_loop: c_int);
}

thread_local!(static MAIN_LOOP_CALLBACK: RefCell<*mut c_void> = RefCell::new(null_mut()));

pub fn set_main_loop_callback<F>(callback: F) where F: FnMut() {
    MAIN_LOOP_CALLBACK.with(|log| {
        *log.borrow_mut() = &callback as *const _ as *mut c_void;
    });

    unsafe { emscripten_set_main_loop(wrapper::<F>, 0, 1); }
}

unsafe extern "C" fn wrapper<F>() where F : FnMut() {
    MAIN_LOOP_CALLBACK.with(|z| {
        let closure = *z.borrow_mut() as *mut F;
        (*closure)();
    });
}
{% endhighlight %}

Finally, change the linker script one more time:
{% highlight bash %}
emcc "-s" "USE_SDL=2" "-o" "cvsolitaire.html" "-02" "-s" "NO_EXIT_RUNTIME=1" $@
{% endhighlight %}

Now rebuild in release mode, ```cargo build --target=asmjs-unknown-emscripten --release``` and open the HTML generated in your current directory. Use Firefox because Chrome does not allow local JS. Alternatively, fire up a simple web server to host the page.

## Bugs
You may have noticed that the colors are off. The green is intentional. The game only uses three suits, and I wanted a better color balance. The blue, however, is a bug. The reds get messed up somewhere along the journey.

Also, not all clicks register, so if you really want to play in your browser, you have to click a few times to get your moves to register. The game is not receiving all mouse events. Any ideas appreciated!

Questions? Comments? I'm gregkatz on github or gregwtmtno on reddit.

[users-guide]: https://users.rust-lang.org/t/compiling-to-the-web-with-rust-and-emscripten/7627
[game]: https://www.github.com/gregkatz/cvsolitaire
[ports]: https://github.com/emscripten-ports
[c-guide]: https://lyceum-allotments.github.io/2016/06/emscripten-and-sdl-2-tutorial-part-1/
[redox]: https://www.redox-os.org
[em-sdl]: https://kripken.github.io/emscripten-site/docs/porting/emscripten-runtime-environment.html#browser-main-loop
[jan-home]: https://fnordig.de/
[triangle-repo]: https://github.com/badboy/rust-triangle-js/blob/master/src/main.rs
[talk]: http://www.hellorust.com/emscripten/
[demo]: {{ site.url }}/assets/game/cvsolitaire.html
[repo]: https://github.com/gregkatz/cvsolitaire/tree/emscripten
