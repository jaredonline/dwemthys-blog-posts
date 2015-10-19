<small style="font-size: 75%"><i>This is Part 1 in a many part series on how to make a roguelike game in Rust. If you're lost, check out the [Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents) to figure out where you should go.</i></small>

# Setup and first pass
Here I'm going to actually get started with some code and setup, but first I want to go over my rough plan. The GitHub tag for this part is [1.1](https://github.com/jaredonline/rust-roguelike/releases/tag/1.1).

### Plan
I'm going to try to condense each concrete topic I learn about into a single post. I'm also going to try and maintain a [GitHub Repository for this project](https://github.com/jaredonline/rust-roguelike), and maintain a tag for every post. The goal there is that you should be able to checkout the repo and follow along by checking out different tags.

## Setup
You'll need a few tools to get this working. I'm on Mac, so all my instructions will be Mac based. To my knowledge, all the libraries work on *nix and Windows. Here's a list as of time of writing:
 - Rust
 - Cargo
 - libtcod
 - tcod-rs

### Rust
Installing Rust is super easy. At the time of writing, [their install instructions](http://doc.rust-lang.org/guide.html#installing-rust) consist of:

```sh
curl -s https://static.rust-lang.org/rustup.sh | sudo sh
```

> A note about nightlies. Rust is under rapid development. As of the time of writing they're on 0.11, and the nightlies were 0.12-pre. This changes really frequently. I'll try to remember to update code with version bumps, but if you run into something that doesn't work, let me know.

You can verify it installed properly by running

```sh
$ which rustc
#=> /usr/local/bin/rustc
```

### Cargo
You get Cargo for free with the above command! Yippee!

You can verify it installed properly by running

```sh
$ which cargo
#=> /usr/local/bin/cargo
```

### libtcod
Installing libtcod on Mac is a little weird. It turns out Mac isn't officially supported anymore... but it seems to work. ~~I followed [these instructions](http://zackhovatter.com/gamedev/2013/11/26/building-libtcod-on-os-x-mavericks.html) and everything was groovy.~~ That link is broken as of 8/21/2014. Fortunately I was able to screen grab the Google Cache. Here's Mac OS X specific install instructions (I believe other platforms have installers):

Install [Homebrew](http://brew.sh/) then install the dependencies:

```sh
brew install sdl mercurial wget upx
```

Then clone the official repo (I put mine in `~/src/`)

```sh
hg clone https://bitbucket.org/jice/libtcod
```

Then make `libtcod` with the following

```sh
cd libtcod
hg checkout 1.5.x
wget https://gist.githubusercontent.com/jaredonline/daf3c5f1ea6c7ca00e29/raw/ae91b3e47bf0de5b772eff882e477d8144cfbaf8/makefile-osx -O makefiles/makefile-osx
wget https://dl.dropboxusercontent.com/u/169446/osx.tar.gz
tar -xzvf osx.tar.gz
make -f makefiles/makefile-osx
make -f makefiles/makefile-samples-linux
```

You can test that it's all working by running

```sh
./samples_c
./samples_cpp
```

### tcod-rs
This took me a *long* time to figure out. There's a project that has Rust bindings to `libtcod` by the name of [tcod-rs](https://github.com/tomassedovic/tcod-rs). The install instructions aren't very helpful.

We aren't going to install `tcod-rs` directly. Instead we'll let Cargo handle that for us, but there is some manual setup involved. We'll address this below.

## First pass
The goal of this is just to get a window to open with the iconic `@` symbol in it. Let's see what we can do.

### Boilerplate
Let's start a new project. I keep all my code in `~/code`, and all my rust projects in `~/code/rust`, so if you see references to that, you can replace them with where ever you keep your code. `cd` to where you want the project to live and run

```sh
cargo new dwemthys --bin
```

This creates a new directory in our current directory that has the boilerplate for a new Rust bin project.

```
.
├── Cargo.toml
└── src
    └── main.rs

1 directory, 2 files
```

Pretty simple. `Cargo.toml` is the file that Cargo reads for meta-data used to build your project. `src/main.rs` contains the `main()` function which is run as our application.

### Add tcod-rs dependency
First thing we need to do is edit the `Cargo.toml` file to add our `tcod-rs` dependency. Open it up and add this to the bottom:

```
[dependencies.tcod]
git = "https://github.com/tomassedovic/tcod-rs.git"
```

This tells Cargo that our project has a dependency named `tcod` that it can find at that GitHub URL. Cargo does the rest of the magic to fetch it, build it and link to it when we build our project.

If you build our project now, it should pass:

```sh
$ cargo build
# Updating git repository `https://github.com/tomassedovic/tcod-rs.git`
# Compiling tcod v0.1.0 (https://github.com/tomassedovic/tcod-rs.git#2f72b1d8)
# Compiling dwemthys v0.0.1 (file:///Users/jmcfarland/code/rust/dwemthys)
```

And it might even run:

```sh
$ cargo run
# Fresh tcod v0.1.0 (https://github.com/tomassedovic/tcod-rs.git#2f72b1d8)
# Fresh dwemthys v0.0.1 (file:///Users/jmcfarland/code/rust/dwemthys)
# Running `target/dwemthys`
# Hello, world!
```

But this is all a farce. If we try to actually use `tcod` we'll get a failure. Let's do that now.  In `src/main.rs` add the following to the top:

```rs
extern crate tcod;
use tcod::Console;
```

And then try running the project again:

```sh
$ cargo run
# A BUNCH OF NASTY OUTPUT!!!

# note: ld: warning: directory not found for option '-L/Users/jmcfarland/code/rust/dwemthys/.rust'
# ld: library not found for -ltcod
# clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

The compiler is complaining it can't find `ltcod`. After a lot of digging, someone pointed me to [this page](http://crates.io/native-build.html) on the Cargo site about native builds. It talks about adding a `build` directive to the `Cargo.toml` file that will run before the compile phase. It passes an env variable, `$OUTPUT_DIR`. That's interesting I thought... but wtf do I do with it? Then I was linked to the [glfw](https://github.com/csherratt/glfw) project which has to link against outside libraries. It has a `.biuld.sh` script it runs to copy some files over. Let's see if copying our `tcod.dylib` files from our source install directory to our `$OUTPUT_DIR` helps.

Create a new file in the root of your project named `.build.sh` (*remember to change the value of `LIBTCOD_SRC_DIR` at the top of this file!*):

```sh
#!/bin/sh

export LIBTCOD_SRC_DIR="/Users/jmcfarland/src/libtcod"
cp LIBTCOD_SRC_DIR/*.dylib $OUT_DIR/
```

And now tell Cargo to run this script before every build by adding a `build` directive in the `[package]` section of our `Cargo.toml` file (right under `authors = `):

```toml
build = "sh .build.sh"
```

Let's try to get the project to run now:

```
$ cargo run
# Fresh tcod v0.1.0 (https://github.com/tomassedovic/tcod-rs.git#2f72b1d8)
# Fresh dwemthys v0.0.1 (file:///Users/jmcfarland/code/rust/dwemthys)
# Running `target/dwemthys`
# Hello, world!
```

Awesome! We're can build a project that dynamically links to `libtcod` now. We practically have a game!

### Make a window show up
For this first part, I'm just going to rip some code straight from the [tcod-rs example](https://github.com/tomassedovic/tcod-rs/blob/master/src/bin/example1.rs), just to see if we can make it work. Right inside your `src/main.rs` file, inside the `main()` function, remove what's there and put this:

```rust
fn main() {
    let mut con = Console::init_root(80, 50, "libtcod Rust tutorial", false);
    let mut exit = false;
    while !(Console::window_closed() || exit) {
        con.clear();
        con.put_char(40, 25, '@', background_flag::Set);
        Console::flush();
        let keypress = Console::wait_for_keypress(true);
        match keypress.key {
            Special(key_code::Escape) => exit = true,
            _ => {}
        }
    }
}
```

At the top of the `src/main.rs` file, we'll need to make sure we're `use`ing the right namespaces:

```rust
use tcod::{Console, background_flag, key_code, Special};
```

Let's see if this runs:

```sh
$ cargo run
# Fresh tcod v0.1.0 (https://github.com/tomassedovic/tcod-rs.git#2f72b1d8)
# Compiling dwemthys v0.0.1 (file:///Users/jmcfarland/code/rust/dwemthys)
# Running `target/dwemthys`
# libtcod 1.6.0
# SDL : cannot load terminal.png
# An unknown error occurred
```

Wtf? What is this `terminal.png`? Who's calling it? Why are we trying to load it? It turns out that the `libtcod` library uses the the `terminal.png` file as a sprite based font and requires it by default. Fortunately I found [this forum post](http://doryen.eptalys.net/forum/index.php?topic=273.0) and [this documentation page](http://doryen.eptalys.net/data/libtcod/doc/1.5.1/html2/console_init_root.html) that helped me sort it out. You need to include `terminal.png` from the root of the project directory. Let's add a simple line to our `.build.sh` script:

```sh
cp $LIBTCOD_SRC_DIR/terminal.png $OUT_DIR/../../../
```

And try to rerun...

```sh
$ cargo run
# Fresh tcod v0.1.0 (https://github.com/tomassedovic/tcod-rs.git#2f72b1d8)
# Compiling dwemthys v0.0.1 (file:///Users/jmcfarland/code/rust/dwemthys)
# Running `target/dwemthys`
# 24 bits font.
# key color : 0 0 0
# character for ascii code 255 is colored
```

![libtcod Rust Tutorial](https://dl.dropboxusercontent.com/u/169446/libtcod-screenshot.png)

Woot! It works. You can get your program to exit by pressing the `esc` key.

## That's it!
We now have a boilerplate project with all our dependencies setup to make an actual game. Next time we'll actually dig into some of the business logic and explore some real Rust code.

## Next
[Part 2: Bring Our Heroine To Life](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-2)

## Table of Contents
[Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents)

## Previous
[Part 0: Why](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust)