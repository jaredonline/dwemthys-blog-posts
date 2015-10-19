<small style="font-size: 75%"><i>This is Part 2 in a many part series on how to make a roguelike game in Rust. If you're lost, check out the [Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents) to figure out where you should go.</i></small>

# Bring Our Heroine To Life

So far our bold rabbit heroine is just an `@` on a screen. To really bring her to life, we need to be able to issue commands (like move, attack, etc) and have the game update the display. Before we get to that a digression...

## Game Loop
At the core of just about every game is the game loop. This is an infinite loop that looks something like this:

```rust
loop {
    check_for_user_input();
    update_game_state();
    render_new_results();
}
```

There's lots written on the internet about the game loop, but a great place to start is [Game Programming Patterns - Game Loop](http://gameprogrammingpatterns.com/game-loop.html). It goes into a lot of great detail about The Game Loop pattern, how to apply it, when to apply it and some of the common gotchas. Fortunately for us, our roguelike doesn't actually need to process events all the time. Our game is turn-based. So we can actually stop processing while we wait for user inputs.

Let's implement our game loop. We'll just take the code from [Part 1](https://svbtle.com/roguelike-tutorial-in-rust-part-1) and make it follow the pattern we want.

```rust
// src/main.rs
fn main() {
    let mut con = Console::init_root(80, 50, "libtcod Rust tutorial", false);
    let mut exit = false;
    while !(Console::window_closed() || exit) {
        // wait for user input
        let keypress = con.wait_for_keypress(true);

        // update game state
        match keypress.key {
            Special(key_code::Escape) => exit = true,
            _ => {}
        }

        // render
        con.clear();
        con.put_char(40, 25, '@', background_flag::Set);
        con.flush();
    }
}
```

That's pretty simple. Just moved some lines around really. We have a problem though... when we launch this we just get a blank white screen.

![White Screen Game](https://dl.dropboxusercontent.com/u/169446/dwemthys/part-2/blank-game.png)

That's not a very exciting game. The problem is that we don't render the game until after user input has been received, so the first rendering happens as the last item of our `loop`. We can fix that by adding the render code outside the `loop`.

```rust
fn main() {
    let mut con = Console::init_root(80, 50, "libtcod Rust tutorial", false);
    let mut exit = false;
    // render
    con.clear();
    con.put_char(40, 25, '@', background_flag::Set);
    con.flush();
    while !(Console::window_closed() || exit) {
      // our game loop 
   }
}
```

That's great... but now we have redundant code. Let's just move that into a function that does the rendering for us.

```rust
fn render(con: &mut Console) {
    con.clear();
    con.put_char(40, 25, '@', background_flag::Set);
    con.flush();
}

fn main() {
    let mut con = Console::init_root(80, 50, "libtcod Rust tutorial", false);
    let mut exit = false;
    // render
    render(&mut con);
    while !(Console::window_closed() || exit) {
        // wait for user input
        let keypress = con.wait_for_keypress(true);

        // update game state
        match keypress.key {
            Special(key_code::Escape) => exit = true,
            _ => {}
        }

        // render
        render(&mut con);
    }
}
```

Cool! Now our code is DRY-er. You should be able to build that and see that it works just as we expect.

A word about some syntax. You'll see that the `render` function has a method signature of `(con: &mut Console)`. That is saying, "I expect one variable, `con`, to be a pointer to a mutable object". And when we call `render()` we prefix the variable being passed in with `&mut` which says, "Take this object, and use a mutable pointer to it". This is only possible because when we instantiated the `con` variable we used the `mut` keyword.

So now we have a game loop. Here's [a link to my commit](https://github.com/jaredonline/rust-roguelike/commit/ce6b9b7d5c4c6b1d0a56f1df75f709ed0608ff25) that gets us up to here.

## Getting Our Heroine to Move
Now that our game loop is in the right format, we can start to do things with it. Let's implement the command to make our rabbit move up on the screen. To do that we need to
 - Add another case to our input match statement
 - Change the x, y coordinates of where our `@` is
 - Change our `render()` function to use our new coordinates

Give it a try before looking at my solution below. (From here on out I'll be removing some sections of code and replacing them with comments if they didn't change, just to save space).

```rust
fn render(con: &mut Console, x: int, y: int) {
    con.clear();
    con.put_char(x, y, '@', background_flag::Set);
    con.flush();
}

fn main() {
    // init window
    let mut charX = 40i;
    let mut charY = 25i;
    // render
    render(&mut con, charX, charY);
    while !(Console::window_closed() || exit) {
        // wait for user input
        // update game state
        match keypress.key {
            Special(key_code::Escape) => exit = true,
            Special(key_code::Up) => {
                if charY >= 1 {
                    charY -= 1;
                }
            },
            _ => {}
        }

        // render
        render(&mut con, charX, charY);
    }
}
```

Did you run into some of the same issues I did? First I didn't know what the `key_code` enum for "up" was, so I looked at the [source definition](https://github.com/tomassedovic/tcod-rs/blob/master/src/lib.rs#L397). Then I forgot that `libtcod` sets the `{0,0}` cord to the top left, so in order to move up you need to *decrease* your Y value. Then I didn't check the out of bounds condition, which allowed me to run the character off screen, and caused a fun crash:

```sh
# task '<main>' failed at 'assertion failed: x >= 0 && y >= 0', src/lib.rs:115
# An unknown error occurred
```

Let's implement down next. This time we only need to:
 - Add a case statement
 - Update our coordinates

```rust
 fn main() {
    let conX = 80i;
    let conY = 50i;
    let mut con = Console::init_root(conX, conY, "libtcod Rust tutorial", false);
    // init variables
    // render
    while !(Console::window_closed() || exit) {
        // wait for user input

        // update game state
        match keypress.key {
            Special(key_code::Escape) => exit = true,
            Special(key_code::Up) => {
                if charY >= 1 {
                    charY -= 1;
                }
            },
            Special(key_code::Down) => {
                if charY < (conY - 1) {
                    charY +=1;
                }
            },
            _ => {}
        }

        // render
    }
}
```

I ran into just one gotcha here. We have to store the height of the window in `conY` so that we can keep our heroine within the window bounds. What I didn't realize at first was that the last Y point in the window is one less than the height (seems obvious in retrospect because we start at point 0).

Now that we have all the pieces in places, let's do both left and right:

```rust
Special(key_code::Left) => {
    if charX >= 1 {
        charX -= 1;
    }
},
Special(key_code::Right) => {
    if charX < (conX - 1) {
        charX += 1;
    }
},
```

This one was super-duper easy. We just added those two conditions. Now our heroine is as free as a bird! She can move in all directions. But she's pretty lonely... let's add a dog to our game world! [Dogs and rabbits can be friends... right?](http://imgur.com/gallery/brzzs) For now, let's just make it wander around the screen randomly.

So, we'll need to do several things here:
 - Add new x and y variables to track it's position
 - Add it to the render method
 - Randomly decide which direction it'll move
 - Update the dog's x and y variables

Let's take them 1 at a time:

```rust
// add the x and y variables
let mut charX = 40i;
let mut charY = 25i;
let mut dogX  = 10i;
let mut dogY  = 10i;
```

Easy enough. Note the `i` at the end of these? Rust has multiple integer types. `int`, `uint`, and more. You can see the [full list here](http://doc.rust-lang.org/rust.html#primitive-types). Next we update our render method:

```rust
fn render(con: &mut Console, x: int, y: int, dogX: int, dogY: int) {
    con.clear();
    con.put_char(x, y, '@', background_flag::Set);
    con.put_char(dogX, dogY, 'd', background_flag::Set);
    con.flush();
}
```

We change our calls to `render`:

```rust
// render
render(&mut con, charX, charY, dogX, dogY);
```

Now let's make our dog move. To do this we'll generate two random ints, from `-1` to `1`. One of these will be our x-offset, and the other will be our y-offset.

```rust
// use the random namespace
use std::rand::Rng;

loop {
        // wait for input
        // update game
        let offset_x = std::rand::task_rng().gen_range(0, 3i) - 1;
        if (dogX + offset_x) > 0 && (dogX + offset_x) < (conX - 1) {
            dogX += offset_x;
        }

        let offset_y = std::rand::task_rng().gen_range(0, 3i) - 1;
        if (dogY + offset_y) > 0 && (dogY + offset_y) < (conY - 1) {
            dogY += offset_y;
        }
        // render
}
```

You should be able to `cargo run` your game at this point and have a `d` that dances around the screen.

### Cleanup
While this all works, the code has gotten pretty messy along the way. Here's my full `src/main.rs` file:

```rust
extern crate tcod;
use tcod::{Console, background_flag, key_code, Special};
use std::rand::Rng;

fn render(con: &mut Console, x: int, y: int, dogX: int, dogY: int) {
    con.clear();
    con.put_char(x, y, '@', background_flag::Set);
    con.put_char(dogX, dogY, 'd', background_flag::Set);
    con.flush();
}

fn main() {
    let conX = 80i;
    let conY = 50i;
    let mut con = Console::init_root(conX, conY, "libtcod Rust tutorial", false);
    let mut exit = false;
    let mut charX = 40i;
    let mut charY = 25i;
    let mut dogX  = 10i;
    let mut dogY  = 10i;
    // render
    render(&mut con, charX, charY, dogX, dogY);
    while !(Console::window_closed() || exit) {
        // wait for user input
        let keypress = con.wait_for_keypress(true);

        // update game state
        match keypress.key {
            Special(key_code::Escape) => exit = true,
            Special(key_code::Up) => {
                if charY >= 1 {
                    charY -= 1;
                }
            },
            Special(key_code::Down) => {
                if charY < (conY - 1) {
                    charY +=1;
                }
            },
            Special(key_code::Left) => {
                if charX >= 1 {
                    charX -= 1;
                }
            },
            Special(key_code::Right) => {
                if charX < (conX - 1) {
                    charX += 1;
                }
            },
            _ => {}
        }

        let offset_x = std::rand::task_rng().gen_range(0, 3i) - 1;
        if (dogX + offset_x) > 0 && (dogX + offset_x) < (conX - 1) {
            dogX += offset_x;
        }

        let offset_y = std::rand::task_rng().gen_range(0, 3i) - 1;
        if (dogY + offset_y) > 0 && (dogY + offset_y) < (conY - 1) {
            dogY += offset_y;
        }

        // render
        render(&mut con, charX, charY, dogX, dogY);
    }
}
```

There are redundancies all over the place. Look at all these `...X`, `...Y` variables we have for starters. Let's wrap them up into a `struct`.

```rust
struct Point {
    x: int,
    y: int
}
```

This way we can encapsulate the fact that these x's and y's belong together. We'll have to change a few method signatures and some code in a few places:

```rust
extern crate tcod;
use tcod::{Console, background_flag, key_code, Special};
use std::rand::Rng;

struct Point {
    x: int,
    y: int
}

fn render(con: &mut Console, c_point: Point, d_point: Point) {
    con.clear();
    con.put_char(c_point.x, c_point.y, '@', background_flag::Set);
    con.put_char(d_point.x, d_point.y, 'd', background_flag::Set);
    con.flush();
}

fn main() {
    let conX = 80i;
    let conY = 50i;
    let mut con = Console::init_root(conX, conY, "libtcod Rust tutorial", false);
    let mut exit = false;
    let mut char_point = Point { x: 40, y: 25 };
    let mut dog_point  = Point { x: 10, y: 10 };
    // render
    render(&mut con, char_point, dog_point);
    while !(Console::window_closed() || exit) {
        // wait for user input
        let keypress = con.wait_for_keypress(true);

        // update game state
        match keypress.key {
            Special(key_code::Escape) => exit = true,
            Special(key_code::Up) => {
                if char_point.y >= 1 {
                    char_point.y -= 1;
                }
            },
            Special(key_code::Down) => {
                if char_point.y < (conY - 1) {
                    char_point.y +=1;
                }
            },
            Special(key_code::Left) => {
                if char_point.x >= 1 {
                    char_point.x -= 1;
                }
            },
            Special(key_code::Right) => {
                if char_point.x < (conX - 1) {
                    char_point.x += 1;
                }
            },
            _ => {}
        }

        let offset_x = std::rand::task_rng().gen_range(0, 3i) - 1;
        if (dog_point.x + offset_x) > 0 && (dog_point.x + offset_x) < (conX - 1) {
            dog_point.x += offset_x;
        }

        let offset_y = std::rand::task_rng().gen_range(0, 3i) - 1;
        if (dog_point.y + offset_y) > 0 && (dog_point.y + offset_y) < (conY - 1) {
            dog_point.y += offset_y;
        }

        // render
        render(&mut con, char_point, dog_point);
    }
}
```

Let's setup another `struct` for the window bounds. It'll be composed of two `Point`s, a min and a max.

```rust
struct Bound {
    min: Point,
    max: Point
}
```

And we can change `conX` and `conY` to this new data structure.

```rust
let window_bounds = Bound { min: Point { x: 0, y: 0 }, max: Point { x: 79, y: 49 } };
```

And everywhere we had `conX` we use `window_bounds.max.x` and `conY` becomes `window_bounds.max.y`.

Now we can do some cool things. First thing we can do is move the logic for offsetting a point into the `Point` struct:

```rust
impl Point {
    fn offset_x(&self, offset: int) -> Point {
        Point { x: self.x + offset, y: self.y }
    }

    fn offset_y(&self, offset: int) -> Point {
        Point { x: self.x, y: self.y + offset }
    }

    fn offset(&self, offset: Point) -> Point {
        Point { x: self.x + offset.x, y: self.y + offset.y }
    }
}
```

Then we can move our logic for checking whether or not a point is inside the window to the `Bounds` struct:

```rust
enum Contains {
    DoesContain,
    DoesNotContain
}

impl Bound {
    fn contains(&self, point: Point) -> Contains {
        if
            point.x >= self.min.x &&
            point.x <= self.max.x &&
            point.y >= self.min.y &&
            point.y <= self.max.y
        {
            DoesContain
        } else {
            DoesNotContain
        }
    }
}
```

And lastly, we can update all of our game loop logic:

```rust
// update game state
let mut offset = Point { x: 0, y: 0 };
match keypress.key {
    Special(key_code::Escape) => exit = true,
    Special(key_code::Up) => {
        offset.y = -1;
    },
    Special(key_code::Down) => {
        offset.y = 1;
    },
    Special(key_code::Left) => {
        offset.x = -1;
    },
    Special(key_code::Right) => {
        offset.x = 1;
    },
    _ => {}
}

match window_bounds.contains(char_point.offset(offset)) {
    DoesContain    => char_point = char_point.offset(offset),
    DoesNotContain => {}
}

let offset_x = std::rand::task_rng().gen_range(0, 3i) - 1;
match window_bounds.contains(dog_point.offset_x(offset_x)) {
    DoesContain    => dog_point = dog_point.offset_x(offset_x),
    DoesNotContain => {}
}

let offset_y = std::rand::task_rng().gen_range(0, 3i) - 1;
match window_bounds.contains(dog_point.offset_y(offset_y)) {
    DoesContain    => dog_point = dog_point.offset_y(offset_y),
    DoesNotContain => {}
}
```

This gives us some pretty good building blocks to work off of. But there's something that still bugs me.  It's centered around our `char_point`, `dog_point` and the `render` function. Imagine the final state of our game. We'll have the character and tons of levels, each with dozens of monsters in it. Are going to add a new `monster_point` for every point? That doesn't seem to make sense to me. Fortunately, Game Programming Patterns has a helpful pattern, [The Update Method](http://gameprogrammingpatterns.com/update-method.html).

## The Update Method
The Update Method is great. It basically says "make every object in your game responsible for updating itself." Makes sense. Let's keep all that logic in one centralized place.

We'll add a new `struct` for tracking the characters in our game. Let's just call it `Character` for now. They'll have a position and a display_char.

```rust
struct Character {
    position:     Point,
    display_char: char
}
```

Now let's replace our point markers with the fully fledged structs, change the `render` method, and replace all instances of `char_point` and `dog_point`:

```rust
fn render(con: &mut Console, ch: Character, dog: Character) {
    con.clear();
    con.put_char(ch.position.x, ch.position.y, ch.display_char, background_flag::Set);
    con.put_char(dog.position.x, dog.position.y, dog.display_char, background_flag::Set);
    con.flush();
}

let mut ch  = Character { position: Point { x: 40, y: 25 }, display_char: '@' };
let mut dog = Character { position: Point { x: 10, y: 10 }, display_char: 'd' };

match window_bounds.contains(ch.position.offset(offset)) {
    DoesContain    => ch.position = ch.position.offset(offset),
    DoesNotContain => {}
}

// etc
```

That previous step didn't actually help solve our problem. We still have to deal with the `ch` variable everywhere and this `dog` variable everywhere. Let's actually implement The Update Method now.

This is by far the biggest refactor we've seen so far. You should try it on your own, and we'll compare notes after you've got it. If you get stuck, the Rust docs, Game Loop Method, and Mozilla IRC #rust channel are great places to ask for help.

Okay, here's my solution:

```rust
extern crate tcod;
use tcod::{Console, background_flag, key_code, Special};
use std::rand::Rng;

struct Point {
    x: int,
    y: int
}

impl Point {
    fn offset_x(&self, offset: int) -> Point {
        Point { x: self.x + offset, y: self.y }
    }

    fn offset_y(&self, offset: int) -> Point {
        Point { x: self.x, y: self.y + offset }
    }

    fn offset(&self, offset: Point) -> Point {
        Point { x: self.x + offset.x, y: self.y + offset.y }
    }
}

struct Bound {
    min: Point,
    max: Point
}

enum Contains {
    DoesContain,
    DoesNotContain
}

impl Bound {
    fn contains(&self, point: Point) -> Contains {
        if
            point.x >= self.min.x &&
            point.x <= self.max.x &&
            point.y >= self.min.y &&
            point.y <= self.max.y
        {
            DoesContain
        } else {
            DoesNotContain
        }
    }
}

struct Game {
    exit:          bool,
    window_bounds: Bound
}

struct Character {
    position:     Point,
    display_char: char
}

impl Character {
    fn new(x: int, y: int, dc: char) -> Character {
        Character { position: Point { x: x, y: y }, display_char: dc }
    }
}

struct NPC {
    position:     Point,
    display_char: char
}

impl NPC {
    fn new(x: int, y: int, dc: char) -> NPC {
        NPC { position: Point { x: x, y: y }, display_char: dc }
    }
}

trait Updates{
    fn update(&mut self, tcod::KeyState, Game);
    fn render(&self, &mut Console);
}

impl Updates for Character {
    fn update(&mut self, keypress: tcod::KeyState, game: Game) {
        let mut offset = Point { x: 0, y: 0 };
        match keypress.key {
            Special(key_code::Up) => {
                offset.y = -1;
            },
            Special(key_code::Down) => {
                offset.y = 1;
            },
            Special(key_code::Left) => {
                offset.x = -1;
            },
            Special(key_code::Right) => {
                offset.x = 1;
            },
            _ => {}
        }

        match game.window_bounds.contains(self.position.offset(offset)) {
            DoesContain    => self.position = self.position.offset(offset),
            DoesNotContain => {}
        }
    }

    fn render(&self, console: &mut Console) {
        console.put_char(self.position.x, self.position.y, self.display_char, background_flag::Set);
    }
}

impl Updates for NPC {
    fn update(&mut self, keypress: tcod::KeyState, game: Game) {
        let offset_x = std::rand::task_rng().gen_range(0, 3i) - 1;
        match game.window_bounds.contains(self.position.offset_x(offset_x)) {
            DoesContain    => self.position = self.position.offset_x(offset_x),
            DoesNotContain => {}
        }

        let offset_y = std::rand::task_rng().gen_range(0, 3i) - 1;
        match game.window_bounds.contains(self.position.offset_y(offset_y)) {
            DoesContain    => self.position = self.position.offset_y(offset_y),
            DoesNotContain => {}
        }
    }

    fn render(&self, console: &mut Console) {
        console.put_char(self.position.x, self.position.y, self.display_char, background_flag::Set);
    }
}

fn render(con: &mut Console, objs: &Vec<Box<Updates>>) {
    con.clear();
    for i in objs.iter() {
        i.render(con);
    }
    con.flush();
}

fn update(objs: &mut Vec<Box<Updates>>, keypress: tcod::KeyState, game: Game) {
    for i in objs.mut_iter() {
        i.update(keypress, game);
    }
}

fn main() {
    let mut game = Game { exit: false, window_bounds: Bound { min: Point { x: 0, y: 0 }, max: Point { x: 79, y: 49 } } };
    let mut con = Console::init_root(game.window_bounds.max.x + 1, game.window_bounds.max.y + 1, "libtcod Rust tutorial", false);
    let c = box Character::new(40, 25, '@') as Box<Updates>;
    let d = box NPC::new(10, 10, 'd') as Box<Updates>;
    let mut objs: Vec<Box<Updates>> = vec![
        c, d
    ];

    render(&mut con, &objs);
    while !(Console::window_closed() || game.exit) {
        // wait for user input
        let keypress = con.wait_for_keypress(true);

        // update game state
        match keypress.key {
            Special(key_code::Escape) => game.exit = true,
            _                         => {}
        }
        update(&mut objs, keypress, game);

        // render
        render(&mut con, &objs);
    }
}
```

If you're thinking, "Wtf?", I don't blame you. This took me a couple hours to figure out.

I'll go through the changes one by one, starting at the top of the file. The first obvious change is the creation of two new structs, `Game` and `NPC`, and the implementation of the `::new()` method on both `NPC` and `Character`. 

```rust
struct Game {
    exit:          bool,
    window_bounds: Bound
}

impl Character {
    fn new(x: int, y: int, dc: char) -> Character {
        Character { position: Point { x: x, y: y }, display_char: dc }
    }
}

struct NPC {
    position:     Point,
    display_char: char
}

impl NPC {
    fn new(x: int, y: int, dc: char) -> NPC {
        NPC { position: Point { x: x, y: y }, display_char: dc }
    }
}
```

The `::new` method is fairly straight-forward. It just returns a new `NPC` or `Character` with the right traits. I split the two out because, according to The Update Method pattern, each type of entity should be responsible for updating itself. In order to have a distinct `#update()` method for the dog and our hero, they have to be different types of entities.

The `Game` struct is just a placeholder for now. I needed to be able to pass around certain pieces of game state and wanted to do them all in one place.

The next change is the introduction of a new `trait`, `Updates`. 

```rust
trait Updates{
    fn update(&mut self, tcod::KeyState, Game);
    fn render(&self, &mut Console);
}
```

Traits are pretty cool in rust. They're like interfaces that can be selectively applied to a type. One of the really powerful things is that you can cast a pointer to the type of one of its traits, like this:

```rust
// create a new character and grab a reference (pointer) to it
let ch = &Character::new()

// cast ch to type &Updates (a reference to an Update type)
ch as &Updates
```

We'll see how useful this is later on. Now that we have our trait, we need to implement it for our two types. Our trait defines two methods, `update` and `render`. First, we'll implement that on our `Character`:

```rust
impl Updates for Character {
    fn update(&mut self, keypress: tcod::KeyState, game: Game) {
        let mut offset = Point { x: 0, y: 0 };
        match keypress.key {
            Special(key_code::Up) => {
                offset.y = -1;
            },
            Special(key_code::Down) => {
                offset.y = 1;
            },
            Special(key_code::Left) => {
                offset.x = -1;
            },
            Special(key_code::Right) => {
                offset.x = 1;
            },
            _ => {}
        }

        match game.window_bounds.contains(self.position.offset(offset)) {
            DoesContain    => self.position = self.position.offset(offset),
            DoesNotContain => {}
        }
    }

    fn render(&self, console: &mut Console) {
        console.put_char(self.position.x, self.position.y, self.display_char, background_flag::Set);
    }
}
```

Look familiar? I pretty much just moved the code from the `main` function into the `update` function for `Character`. I'll do the same thing for `NPC`:

```rust
impl Updates for NPC {
    fn update(&mut self, keypress: tcod::KeyState, game: Game) {
        let offset_x = std::rand::task_rng().gen_range(0, 3i) - 1;
        match game.window_bounds.contains(self.position.offset_x(offset_x)) {
            DoesContain    => self.position = self.position.offset_x(offset_x),
            DoesNotContain => {}
        }

        let offset_y = std::rand::task_rng().gen_range(0, 3i) - 1;
        match game.window_bounds.contains(self.position.offset_y(offset_y)) {
            DoesContain    => self.position = self.position.offset_y(offset_y),
            DoesNotContain => {}
        }
    }

    fn render(&self, console: &mut Console) {
        console.put_char(self.position.x, self.position.y, self.display_char, background_flag::Set);
    }
}
```

Now that we've contained all the logic for updating and rendering a unit to that unit, we have all the pieces in place to finish implementing The Update Method pattern. Jump down the file to where we initialize our rabbit and dog, and you'll see I've changed them to:

```rust
let c = box Character::new(40, 25, '@') as Box<Updates>;
let d = box NPC::new(10, 10, 'd') as Box<Updates>;
let mut objs: Vec<Box<Updates>> = vec![
    c, d
];
```

There's some confusing syntax there, I'll walk through it. First, we create two  values, `c` and `d`. One is a `Character` and the other a `Dog`, but they're both using the `as` keyword to cast them to `Update`. Also notice the `box` and `Box` syntax, which basically says, this is a heap allocated variable. Then we initialize a new value, `objs` to be type `Vec<Box<Updates>>`. That basically says "A vector of boxes of type Updates". This was one of the more difficult parts for me to wrap my head around. The Rust docs have a good [writeup on pointers in Rust](http://doc.rust-lang.org/guide-pointers.html).

Now we have a vector of all our objects that need to be updated and rendered in our game loop, let's change our `render` method and introduce a new `update` method:

```rust
fn render(con: &mut Console, objs: &Vec<Box<Updates>>) {
    con.clear();
    for i in objs.iter() {
        i.render(con);
    }
    con.flush();
}

fn update(objs: &mut Vec<Box<Updates>>, keypress: tcod::KeyState, game: Game) {
    for i in objs.mut_iter() {
        i.update(keypress, game);
    }
}
```

In the `render` method we now loop through our `objs` vector and call render on each item individually, and in our `update` method we do the same thing. Note the `mut_iter()` in the `update` method. This is because the `update` method needs to have a mutable reference to `self` to work. Getting my references and mutable state correct is the hardest part of learning Rust (for me) so far.

That was a really big but really powerful change-set. Here's [a commit to where we just ended up](https://github.com/jaredonline/rust-roguelike/commit/7e2830f86324eeb4961266bc1fa5d7a5ef595554).

## Structure
There's one last thing I want to do before ending this tutorial. Right now everything is in a giant file, `src/main.rs`. That's silly, we have directories for a reason. Let's use them. We'll use Rust modules to move different pieces of code into different files and include them in our main file. We'll start with `Point` because it's the smallest.

The first step is to create a new file `src/lib.rs`. This file will act like a manifest, pointing to all the other modules we create. For now, leave it empty. Our directory structure looks like this now:

```
.
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── lib.rs
│   └── main.rs
```

Now create a new folder, `src/util/` and a new file in it, `src/util/mod.rs`. Copy the following from `src/main.rs` into `src/util/mod.rs` so it looks like this:

```rust
struct Point {
    x: int,
    y: int
}

impl Point {
    fn offset_x(&self, offset: int) -> Point {
        Point { x: self.x + offset, y: self.y }
    }

    fn offset_y(&self, offset: int) -> Point {
        Point { x: self.x, y: self.y + offset }
    }

    fn offset(&self, offset: Point) -> Point {
        Point { x: self.x + offset.x, y: self.y + offset.y }
    }
}
```

Now in `src/lib.rs` add a single line:

```rust
pub mod util;
```

And finally, add to the top of `src/main.rs`:

```rust
// below extern create tcod;
extern crate dwemthys;

// after all other extern crate lines
use dwemthys::util::Point;
```

This tells our `main.rs` file to use the `dwemthys` crate (a Rust module) and the `util` submodule and the `Point` struct in the top level namespace.

If you try to build right now you'll get a whole ton of errors, and they should all be complaining about accessing private fields of `Point`. This is because everything in a module is `private` by default, and must explicitly marked as public. So we add a bunch of `pub` in the `src/util/mod.rs` file:

```rust
pub struct Point {
    pub x: int,
    pub y: int
}

impl Point {
    pub fn offset_x(&self, offset: int) -> Point {
        Point { x: self.x + offset, y: self.y }
    }

    pub fn offset_y(&self, offset: int) -> Point {
        Point { x: self.x, y: self.y + offset }
    }

    pub fn offset(&self, offset: Point) -> Point {
        Point { x: self.x + offset.x, y: self.y + offset.y }
    }
}
```

Things that have to be marked `pub` are: `struct` definitions, traits (like `x` and `y` in our `struct` definition) and `fn`s. Now if you build it should all just work. Let's do the same thing with `Bound` and `Contains`:

```rust
// src/util/mod.rs
pub enum Contains {
    DoesContain,
    DoesNotContain
}

pub struct Bound {
    pub min: Point,
    pub max: Point
}

impl Bound {
    pub fn contains(&self, point: Point) -> Contains {
        if
            point.x >= self.min.x &&
            point.x <= self.max.x &&
            point.y >= self.min.y &&
            point.y <= self.max.y
        {
            DoesContain
        } else {
            DoesNotContain
        }
    }
}
```

And we modify our `use` statement in `src/main.rs` to look like this: 

```rust
use dwemthys::util::{Point, Bound}
```

If you try to build at this point you'll get some interesting errors:

```
/Users/jmcfarland/code/rust/dwemthys/src/main.rs:36:13: 36:27 error: unreachable pattern [E0001] (pass `--explain E0001` to see a detailed explanation)
/Users/jmcfarland/code/rust/dwemthys/src/main.rs:36             DoesNotContain => {}
                                                                ^~~~~~~~~~~~~~
/Users/jmcfarland/code/rust/dwemthys/src/main.rs:42:13: 42:27 error: unreachable pattern [E0001] (pass `--explain E0001` to see a detailed explanation)
/Users/jmcfarland/code/rust/dwemthys/src/main.rs:42             DoesNotContain => {}
                                                                ^~~~~~~~~~~~~~
/Users/jmcfarland/code/rust/dwemthys/src/main.rs:83:13: 83:27 error: unreachable pattern [E0001] (pass `--explain E0001` to see a detailed explanation)
/Users/jmcfarland/code/rust/dwemthys/src/main.rs:83             DoesNotContain => {}
```

Apparently our compiler thinks we can't ever get to the `DoesNotContain` case, even though I'm pretty sure we can still get there. The solution to this is a little in-elegant, but we need to raise `DoesNotContain` and `DoesContain` to the top-level namespace (or qualify them):

```rust
use dwemthys::util::{Point, Bound, DoesContain, DoesNotContain};
```

Next we'll move the `Game` struct to it's own module. This one is fairly straight forward.  Note the `use util::Bound` at the top as `Game` uses the `Bound` struct:

```rust
use util::Bound;

pub struct Game {
    pub exit:          bool,
    pub window_bounds: Bound
}
```

Then we we add it to the `src/lib.rs` file:

```
pub mod game;
```

And we should be able to build just fine. Next let's try to move out one of our move complicated structs, `Character`. This one requires creating two mods actually, because both `NPC` and `Character` rely on the `Update` trait, so let's start there:

```rust
// src/traits/mod.rs
extern crate tcod;
use self::tcod::{Console};
use game::Game;

pub trait Updates {
    fn update(&mut self, tcod::KeyState, Game);
    fn render(&self, &mut Console);
}
```

Interestingly, it looks like the methods in a `trait` don't need to be marked as `pub`, just the top-level trait itself. Once we have that, we can add `pub mod traits;` to our `src/lib.rs` file and `use dwemthys::traits::Updates` to our `src/main.rs` file.

Now let's do the same thing for `Character`:

```rust
extern crate tcod;
use self::tcod::{Console, background_flag, key_code, Special};

use traits::Updates;
use util::{Point, DoesContain, DoesNotContain};
use game::Game;

pub struct Character {
    pub position:     Point,
    pub display_char: char
}

impl Character {
    pub fn new(x: int, y: int, dc: char) -> Character {
        Character { position: Point { x: x, y: y }, display_char: dc }
    }
}

impl Updates for Character {
    fn update(&mut self, keypress: tcod::KeyState, game: Game) {
        let mut offset = Point { x: 0, y: 0 };
        match keypress.key {
            Special(key_code::Up) => {
                offset.y = -1;
            },
            Special(key_code::Down) => {
                offset.y = 1;
            },
            Special(key_code::Left) => {
                offset.x = -1;
            },
            Special(key_code::Right) => {
                offset.x = 1;
            },
            _ => {}
        }

        match game.window_bounds.contains(self.position.offset(offset)) {
            DoesContain    => self.position = self.position.offset(offset),
            DoesNotContain => {}
        }
    }

    fn render(&self, console: &mut Console) {
        console.put_char(self.position.x, self.position.y, self.display_char, background_flag::Set);
    }
}
```

This is pretty straight forward at this point. Include all the things our module relies on, make sure we mark the right functions and definitions as public. Nothing new here [=

Let's do the same thing with NPC:

```rust
extern crate tcod;
use self::tcod::{Console, background_flag};

use traits::Updates;
use util::{Point, DoesContain, DoesNotContain};
use game::Game;

use std;
use std::rand::Rng;

pub struct NPC {
    position:     Point,
    display_char: char
}

impl NPC {
    pub fn new(x: int, y: int, dc: char) -> NPC {
        NPC { position: Point { x: x, y: y }, display_char: dc }
    }
}

impl Updates for NPC {
    fn update(&mut self, keypress: tcod::KeyState, game: Game) {
        let offset_x = std::rand::task_rng().gen_range(0, 3i) - 1;
        match game.window_bounds.contains(self.position.offset_x(offset_x)) {
            DoesContain    => self.position = self.position.offset_x(offset_x),
            DoesNotContain => {}
        }

        let offset_y = std::rand::task_rng().gen_range(0, 3i) - 1;
        match game.window_bounds.contains(self.position.offset_y(offset_y)) {
            DoesContain    => self.position = self.position.offset_y(offset_y),
            DoesNotContain => {}
        }
    }

    fn render(&self, console: &mut Console) {
        console.put_char(self.position.x, self.position.y, self.display_char, background_flag::Set);
    }
}
```

Same sort of deal. Copy a bunch of code over, make sure we have the right `use` statements, mark everything as `pub` and we're good to go. At this point you should have a directory structure that looks like this:

```
.
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── character
│   │   └── mod.rs
│   ├── game
│   │   └── mod.rs
│   ├── lib.rs
│   ├── main.rs
│   ├── npc
│   │   └── mod.rs
│   ├── traits
│   │   └── mod.rs
│   └── util
│       └── mod.rs
```

And a `src/main.rs` that looks like this:

```rust
extern crate tcod;
extern crate dwemthys;

use dwemthys::util::{Point, Bound};
use dwemthys::traits::Updates;
use dwemthys::game::Game;
use dwemthys::character::Character;
use dwemthys::npc::NPC;

use tcod::{Console, key_code, Special};

fn render(con: &mut Console, objs: &mut Vec<Box<Updates>>) {
    con.clear();
    for i in objs.iter() {
        i.render(con);
    }
    con.flush();
}

fn update(objs: &mut Vec<Box<Updates>>, keypress: tcod::KeyState, game: Game) {
    for i in objs.mut_iter() {
        i.update(keypress, game);
    }
}

fn main() {
    let mut game = Game { exit: false, window_bounds: Bound { min: Point { x: 0, y: 0 }, max: Point { x: 79, y: 49 } } };
    let mut con = Console::init_root(game.window_bounds.max.x + 1, game.window_bounds.max.y + 1, "libtcod Rust tutorial", false);
    let c = box Character::new(40, 25, '@') as Box<Updates>;
    let d = box NPC::new(10, 10, 'd') as Box<Updates>;
    let mut objs: Vec<Box<Updates>> = vec![
        c, d
    ];

    render(&mut con, &objs);
    while !(Console::window_closed() || game.exit) {
        // wait for user input
        let keypress = con.wait_for_keypress(true);

        // update game state
        match keypress.key {
            Special(key_code::Escape) => game.exit = true,
            _                         => {}
        }
        update(&mut objs, keypress, game);

        // render
        render(&mut con, &objs);
    }
}
```

Nice! It only has three methods. I think it's safe to stop there for now.

## Conclusion
Holy crap! That was a lot more work than I thought it was going to be. I hope you made it this far without too much trouble. There's a lot I'd like to change about our current implementation, but I think we're in a good state. We did what we set out to do, we made our heroine come alive. Here's [a tag to the end state after this part](https://github.com/jaredonline/rust-roguelike/releases/tag/2.1).

## Next
[Part 3: Combat!](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-3)
## Previous

## Table of Contents
[Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents)

## Previous
[Part 1: Setup and First Pass](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-1)