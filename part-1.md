<small style="font-size: 75%"><i>This is Part 1 in a many part series on how to make a roguelike game in Rust. If you're lost, check out the [Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents) to figure out where you should go.</i></small>

# Setup and first pass
Here I'm going to actually get started with some code and setup, but first I want to go over my rough plan. The GitHub tag for this part is [1.3](https://github.com/jaredonline/rust-roguelike/releases/tag/1.3).

### Plan
I'm going to try to condense each concrete topic I learn about into a single post. I'm also going to try and maintain a [GitHub Repository for this project](https://github.com/jaredonline/rust-roguelike), and maintain a tag for every post. The goal there is that you should be able to checkout the repo and follow along by checking out different tags.

We're going to use a couple of pre-made building blocks to help us along our way, but I'm going to try and use as little as possible. The goal of this project is to learn. We'll be using [`libtcod`](http://roguecentral.org/doryen/libtcod/) to handle the input/output functionality. It can do *a lot* more than that, but for now I plan on only using its I/O.

## Setup
You'll need a few tools to get this working. I'm on Mac, so all my instructions will be Mac based. To my knowledge, all the libraries work on *nix and Windows. Here's a list as of time of writing:
 - Rust
 - Cargo
 - tcod-rs
 - libtcod

### Rust
Installing Rust is super easy. At the time of writing, [their install instructions](http://doc.rust-lang.org/guide.html#installing-rust) consist of:

```sh
curl -s https://static.rust-lang.org/rustup.sh | sudo sh
```

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

### tcod-rs
There's a project that has Rust bindings to [`libtcod`](http://roguecentral.org/doryen/libtcod/) by the name of [tcod-rs](https://github.com/tomassedovic/tcod-rs). We aren't going to install `tcod-rs` directly. Instead we'll let Cargo handle that for us, but there is some manual setup involved. We'll address this below.

### libtcod
Once upon a time you had to build `libtcod` on your own if you were using `tcod-rs`, but that's not the case anymore. I'm leaving this here with a [link]() to legacy instructions just for historic purposes.

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
# Updating registry `https://github.com/rust-lang/crates.io-index`
# Compiling tcod-sys v3.0.0 (https://github.com/tomassedovic/tcod-rs.git#59826643)
# Compiling libc v0.1.10
# Compiling bitflags v0.1.1
# Compiling lazy_static v0.1.15
# Compiling tcod v0.8.0 (https://github.com/tomassedovic/tcod-rs.git#59826643)
# Compiling dwemthys v0.1.0 (file:///Users/jmcfarland/code/rust/dwemthys)
```

And it should even run!

```sh
$ cargo run
# Running `target/debug/dwemthys`
# Hello, world!
```

Awesome! We're can build a project that dynamically links to `libtcod` now. We practically have a game!

### Make a window show up
For this first part, I'm just going to rip some code straight from the [tcod-rs example](https://github.com/tomassedovic/tcod-rs/blob/master/examples/keyboard.rs), just to see if we can make it work. Right inside your `src/main.rs` file, inside the `main()` function, remove what's there and put this:

```rust
fn main() {
    let mut con = RootConsole::initializer()
        .size(80, 50)
        .title("libtcod Rust tutorial")
        .init();

    let mut exit = false;
    while !(con.window_closed() || exit) {
        con.clear();
        con.put_char(40, 25, '@', BackgroundFlag::Set);
        con.flush();
        let keypress = con.wait_for_keypress(true);

        match keypress {
            Key { code: Escape, .. } => exit = true,
            _ => {}
        }
    }
}
```

At the top of the `src/main.rs` file, we'll need to make sure we're `use`ing the right namespaces:

```rust
extern crate tcod;
use tcod::{Console, RootConsole, BackgroundFlag,};
use tcod::input::Key;
use tcod::input::KeyCode::{Escape,};
```

Let's see if this runs:

```sh
$ cargo run
# Compiling dwemthys v0.1.0 (file:///Users/jmcfarland/code/rust/dwemthys)
# Running `target/debug/dwemthys`
# libtcod 1.5.2
# SDL : cannot load terminal.png
# An unknown error occurred

# To learn more, run the command again with --verbose.
```

Wtf? What is this `terminal.png`? Who's calling it? Why are we trying to load it? It turns out that the `libtcod` library uses the the `terminal.png` file as a sprite based font and requires it by default. Fortunately I found [this forum post](http://doryen.eptalys.net/forum/index.php?topic=273.0) and [this documentation page](http://doryen.eptalys.net/data/libtcod/doc/1.5.1/html2/console_init_root.html) that helped me sort it out. You need to include `terminal.png` from the root of the project directory. I've uploaded [the one I'm using](https://dl.dropboxusercontent.com/u/169446/dwemthys/part-1/terminal.png) to Dropbox. Just download it into the root directory on your machine.

And try to rerun...

```sh
$ cargo run
# Running `target/debug/dwemthys`
# 24 bits font.
# key color : 0 0 0
# character for ascii code 255 is colored
```

![libtcod Rust Tutorial](https://dl.dropboxusercontent.com/u/169446/dwemthys/part-1/it_works.png)

Woot! It works. You can get your program to exit by pressing the `esc` key or closing the window.

## That's it!
We now have a boilerplate project with all our dependencies setup to make an actual game. Next time we'll actually dig into some of the business logic and explore some real Rust code.

## Next
[Part 2: Bring Our Heroine To Life](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-2)

## Table of Contents
[Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents)

## Previous
[Part 0: Why](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust)