<small style="font-size: 75%"><i>This is Part 3 in a many part series on how to make a roguelike game in Rust. If you're lost, check out the [Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents) to figure out where you should go.</i></small>

# Combat!

Things are progressing nicely! We have rabbit that we can control, and she has a sweet dog that wanders aimlessly. The goal of this part will be to get some enemies on the screen and get our rabbit to fight them.

But before we get started on that, we have some chores to do. There's some things about our code base that are a little sub-optimal right now. 

## int or i32
First, and probably the easiest, it's good practice specify our integer lengths whenever possible. That means not using `int` in favor of `i32` or `u32`.  We only use `int` in a few places, and I think `i32` will work for all of them, so go ahead and make that change in `character/mod.rs`, `npc/mod.rs` and `util/mod.rs`. I ran into a small snag when making this change. After changing everything from `int` to `i32`, the project wouldn't build anymore:

```
/Users/jmcfarland/code/rust/dwemthys/src/npc/mod.rs:38:43: 38:58 error: mismatched types: expected `int` but found `i32` (expected int but found i32)
/Users/jmcfarland/code/rust/dwemthys/src/npc/mod.rs:38         console.put_char(self.position.x, self.position.y, self.display_char, background_flag::Set);
```

Our `tcod-rs` library was looking for `ints` but we sent it `i32s`. This is pretty easy to fix by changing that line to:

```rust
console.put_char(self.position.x as int, self.position.y as int, self.display_char, background_flag::Set);
```

Notice the `as int`, which is how you typecast in Rust. There should be five or six places you have to change this.

## Cats
Next let's talk about a small bug our program has and how we want to go about fixing it. Change the section in `src/main.rs` where we initialize our heroine and our dog to add a new `NPC`. Let's make a cat:

```rust
let c = box Character::new(40, 25, '@') as Box<Updates>;
let d = box NPC::new(10, 10, 'd') as Box<Updates>;
let ct = box NPC::new(40, 25, 'c') as Box<Updates>;
let mut objs: Box<Vec<Updates>> = vec![
    d, c, ct
];
```

Build and run the game, and take a look at the results:

![Overlap Bug](https://dl.dropboxusercontent.com/u/169446/dwemthys/part-3/overlap-bug.png)

What happened to our heroine?! She's been turned into a cat! Actually, she's just hiding behind the cat, but because we rendered the cat after our heroine the cat overwrote the heroine. Remember, this is a roguelike, we can only display one character in any given position. We could solve this naively and just add the `Character` to the array last:

```rust
let c = box Character::new(40, 25, '@') as Box<Updates>;
let d = box NPC::new(10, 10, 'd') as Box<Updates>;
let ct = box NPC::new(40, 25, 'c') as Box<Updates>;
let mut objs: Box<Vec<Updates>> = vec![
    d, ct, c
];
```

But that's pretty brittle, and it makes dynamically adding and removing NPCs from our array much more difficult; we'd have to make sure we always leave our heroine as the last element. Instead I want to sort the game into layers. Let's say that all NPCs and all characters will be in their own layers. Because we'll only ever have one character, it no longer needs to be in an array.

```rust
fn render(con: &mut Console, npcs: &Vec<Box<Updates>>, c: Character) {
    con.clear();
    for i in objs.iter() {
        i.render(con);
    }
    c.render(con);
    con.flush();
}

let mut c = Character::new(40, 25, '@');
let d = box NPC::new(10, 10, 'd') as Box<Updates>;
let ct = box NPC::new(40, 25, 'c') as Box<Updates>;
let mut npcs: Box<Vec<Updates>> = vec![
    d, ct
];

render(&mut con, &npcs, c);
```

Notice that we're not longer casting `c` to `Box<Updates>` (in fact it's no longer a `Box` at all) and that we're declaring it as `mut`. Also notice I changed the name of the vector to `npcs` from `objs` to better reflect what it holds.

Once we change all the variables from `objs` to `npcs` the game will build but there's a problem. Our character is no longer responding to input. This is bad news. We took her out of the array that we pass to `#update` so now we never update her state. For now, let's solve this by changing `#update` just like we changed `#render`. I'm not sure if I like this solution, but let's see where it takes us.

```rust
fn update(objs: &Vec<&mut Updates>, c: &mut Character, keypress: tcod::KeyState, game: Game) {
    c.update(keypress, game);
    for &mut i in objs.iter() {
        i.update(keypress, game);
    }
}

update(&npcs, &mut c, keypress, game);
```

Notice I have the character updating before I update any of the NPCs. This is so that when the NPCs update they have the latest information about where the character is and what it's state is before making their moves.

*Side-note*: this change lets us make fix a warning that's been bugging me for awhile. Have you been getting this every time you build your project?

```
/Users/jmcfarland/code/rust/dwemthys/src/npc/mod.rs:23:26: 23:34 warning: unused variable: `keypress`, #[warn(unused_variable)] on by default
/Users/jmcfarland/code/rust/dwemthys/src/npc/mod.rs:23     fn update(&mut self, keypress: tcod::KeyState, game: Game) {
```

This is happening because we're sending the NPCs the `keypress` variable, but their version of `#update` never uses it. Now that we've actually separated the `Character` and the `NPC` from the array, we can remove that variable. This refactor is pretty straight forward but requires changes in a lot of places. Try to make it yourself before looking at my solution.

```rust
// src/main.rs
// remove the keypress variable from the call to update
for &mut i in objs.iter() {
    i.update(game);
}

// src/traits/mod.rs
// remove the keypress variable from the method signature
fn update(&mut self, Game);

// src/npc/mod.rs
// remove the keypress variable from the method signature
fn update(&mut self, game: Game) {

// src/character/mod.rs
// 1) remove the `use traits::Update`
// 2) move `update` and `render` into the base struct `impl` and 
//    mark the public
impl Character {
    pub fn update(&mut self, keypress: tcod::KeyState, game: Game) {
         // do stuff
    }

    pub fn render(&self, console: &mut Console) {
        // do stuff
    }
}
```

## Component Pattern
Here's the last bit of cleanup before we start on combat. We have *a lot* of coupling going on here. Especially in our `Character` implementation. It's coupled very tightly to `tcod-rs` in the way we handle input, and even more generically it's coupled to the fact that we take input from the keyboard. What if we wanted to switch to the mouse? That shouldn't concern our `Character` class.

Introducing the [Component Pattern](http://gameprogrammingpatterns.com/component.html). This pattern is pretty cool and a bit tricky, so give it a read through before we start. Ready? Cool. Me too.

Let's start off by moving our rendering into a component because it's the most simple. 

First, let's make a `TcodRenderingComponent` in our `src/main.rs` file. We'll move the file eventually, but I find it's nice to keep all my code in a single file while I'm still sketching it out. Here's my component:

```rust
struct TcodRenderingComponent {
    console: Console
}

impl TcodRenderingComponent {
    fn before_render_new_frame(&mut self) {
        self.console.clear();
    }

    fn render_object(&mut self, position: Point, symbol: char) {
        self.console.put_char(position.x as int, position.y as int, symbol, background_flag::Set);
    }

    fn after_render_new_frame(&mut self) {
        self.console.flush();
    }
}
```

Notice that we moved the `console.clear()` and `console.flush()` calls into our struct. This is because once we initialize our rendering component with the console, it's "borrows" ownership it and won't let it go until the struct is deconstructed. Because our rendering component is around for the lifetime of our game, we need to move all calls to our console into the component. This is actually a great way to make sure we're decoupled. We can't accidentally make calls to the console from outside our component anymore.

I ran into a snag trying actually implement this and couldn't figure out how to import my new struct from the `src/main.rs` file into the `character/mod.rs` and `npc/mod.rs` file, so I moved it to it's own module at this point (and called it `rendering`). 

```rust
extern crate tcod;
use self::tcod::{Console, background_flag};

use util::{Point};

pub struct TcodRenderingComponent {
    console: Console
}

impl TcodRenderingComponent {
    pub fn before_render_new_frame(&mut self) {
        self.console.clear();
    }

    pub fn render_object(&mut self, position: Point, symbol: char) {
        self.console.put_char(position.x as int, position.y as int, symbol, background_flag::Set);
    }

    pub fn after_render_new_frame(&mut self) {
        self.console.flush();
    }
}
```

Don't forget to add `pub mod rendering;` to your `src/lib.rs` file. Let's initialize a new `TcodRenderingComponent` in our `main` function and see what happens:

```rust
let mut rendering_component = box TcodRenderingComponent { console: con };
```

Go ahead and build the project. You should get a ton of errors that look like this:

```
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:42:24: 42:27 error: use of moved value: `con`
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:42         let keypress = con.wait_for_keypress(true);
                                                                                   ^~~
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:39:12: 39:15 note: `con` moved here because it has type `tcod::Console`, which is non-copyable (perhaps you meant to use clone()?)
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:39     render(con, &npcs, c);
```

The compiler is complaining because when we instantiate our rendering component with `con`, we're "moving" ownership of `con` to the rendering component. As long as that rendering component is still around, it owns `con` and nothing else is allowed to use it. To fix this we'll have to move all calls to `con` into the rendering component. The only one we haven't address so far is `#wait_for_keypress`:

```rust
use self::tcod::{Console, background_flag, KeyState};

pub fn wait_for_keypress(&mut self) -> KeyState {
    self.console.wait_for_keypress(true)
}
```

Now we'll have to find all our references to `con` and replace them with calls to `rendering_component`. Let's take a look at the change to our `#render` function in `main.rs`:

```rust
fn render(rendering_component: &mut Box<TcodRenderingComponent>, npcs: &Vec<Box<Updates>>, c: Character) {
    rendering_component.before_render_new_frame();
    for i in npcs.iter() {
        i.render(rendering_component);
    }
    c.render(rendering_component);
    rendering_component.after_render_new_frame();
}
```

It looks pretty similar, but we pass around the `rendering_component` instead of the `console`. Nice! Let's make the changes to `NPC` and `Character` (which are identical):

```rust
pub fn render(&self, rendering_component: &mut Box<TcodRenderingComponent>) {
    rendering_component.render_object(self.position, self.display_char);
}
```

Now let's update our main loop to reflect all the changes we just made:

```rust
render(&mut rendering_component, &npcs, c);
while !(Console::window_closed() || game.exit) {
    // wait for user input
    let keypress = rendering_component.wait_for_keypress();

    // update game state
    match keypress.key {
        Special(key_code::Escape) => game.exit = true,
        _                         => {}
    }
    update(&mut npcs, &mut c, keypress, game);

    // render
    render(&mut rendering_component, &npcs, c);
}
```

Go ahead and try to build the project. I ran into a bunch of different compiler issues around making things `pub` or not, what I imported where, stuff like that. You should be able to follow the errors to get everything to build. When you do you might see a lot of warnings like this:

```
/Users/jmcfarland/code/rust/dwemthys/src/npc/mod.rs:2:27: 2:42 warning: unused import, #[warn(unused_imports)] on by default
```

This is rust telling us we imported something but we're not using it. Cool! The compiler has our back! The only thing that should be containing references to the `tcod::Console` and `tcod::background_flag` now are within our rendering component, so we can remove those imports from *everywhere else* (except where we instantiate the component). Yay decoupling!

### Actually making components generic
Say we wanted to move from `libtcod` to `ncurses` (lol) for our rendering. What would have to do? Well... 
 - introduce a new component, `NcursesRenderingComponent`
 - initialize the `NcursesRenderingComponent` in `main`
 - change references in all our method signatures to use `NcursesRenderingComponent` instead of `TcodRenderingComponent`

That's a lot more than we want. We really want to just do the top two, right? Otherwise we're not really decoupled. We still have references to `tcod` floating around our codebase... they're just a little more obscure now. Let's actually get rid of them. How? By introducing a new generic type. We'll use a `trait` for this:

```rust
pub trait RenderingComponent {
    fn before_render_new_frame(&mut self);
    fn render_object(&mut self, Point, char);
    fn after_render_new_frame(&mut self);
    fn wait_for_keypress(&mut self) -> KeyState;
}

impl RenderingComponent for TcodRenderingComponent {
    fn before_render_new_frame(&mut self) {
        self.console.clear();
    }

    fn render_object(&mut self, position: Point, symbol: char) {
        self.console.put_char(position.x as int, position.y as int, symbol, background_flag::Set);
    }

    fn after_render_new_frame(&mut self) {
        self.console.flush();
    }

    fn wait_for_keypress(&mut self) -> KeyState {
        self.console.wait_for_keypress(true)
    }
}
```

As far as I know, that simple change shouldn't have any effect on our code, but if you try to build the game now, you'll get the following:

```
/Users/jmcfarland/code/rust/dwemthys-working/src/character/mod.rs:43:29: 43:76 error: type `&mut rendering::TcodRenderingComponent` does not implement any method in scope named `render_object`
/Users/jmcfarland/code/rust/dwemthys-working/src/character/mod.rs:43         rendering_component.render_object(self.position, self.display_char);
                                                                                                 ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/Users/jmcfarland/code/rust/dwemthys-working/src/npc/mod.rs:36:29: 36:76 error: type `&mut rendering::TcodRenderingComponent` does not implement any method in scope named `render_object`
/Users/jmcfarland/code/rust/dwemthys-working/src/npc/mod.rs:36         rendering_component.render_object(self.position, self.display_char);
                                                                                           ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
error: aborting due to 2 previous errors
Could not compile `dwemthys`.
```

I'm not sure if this is a bug or intended behavior, but once we tell the compiler that `TcodRenderingComponent` implements `RenderingComponent` everything goes bananas. There's a weird pattern to get around this. First we'll make a new static method as part of the `RenderingComponent` trait, `::new()` which has a return type of `Self`. Then, when implementing that on the `TcodRenderingComponent` we give it a return type of `TcodRenderingComponent`. This will act as our initializer, so we'll stop manually initializing the `TcodRenderingComponent`. Let's see what this looks like:

```rust
pub trait RenderingComponent {
    fn new(Console) -> Self;
    fn before_render_new_frame(&mut self);
    fn render_object(&mut self, Point, char);
    fn after_render_new_frame(&mut self);
    fn wait_for_keypress(&mut self) -> KeyState;
}

// in the implementation
fn new(console: Console) -> TcodRenderingComponent {
    TcodRenderingComponent {
        console: console
    }
}
```

Then we initialize the component in a sort of funky way:

```rust
let mut rendering_component : Box<TcodRenderingComponent> = box RenderingComponent::new(con);
```

Then we can change references in `Character` and `NPC` from `TcodRenderingComponent` to just `RenderingComponent` and we've now really decoupled our rendering engine. Unfortunately, at this point I hit another snag which I don't fully understand. I can't get the Rust compiler to like the types I'm using. If I give method signature's a type of `&mut RenderingComponent` they complain that a `Box` isn't an `&-ptr`. If the method signatures have `Box<RenderingComponent>` I get compiler errors saying `no implementation of trait RenderingComponent found for TcodRenderingComponent`. At this point I decided I'm just tired of passing around all these variables and I want to move all this logic into the Game object. Here we go on another big refactor.

### Moving Logic to Game
First let's make `rendering_component` a field on `Game`. This makes sense because we'll only ever use one rendering component at a time in the game. To do this we need to add a lifetime variable to `Game`. The rust docs will do a much better job explaining them than I will, so here's the [Rust Guide to Lifetimes](http://doc.rust-lang.org/guide-lifetimes.html).

```rust
pub struct Game<'a> {
    pub exit:                bool,
    pub window_bounds:       Bound,
    pub rendering_component: Box<RenderingComponent + 'a>
}
```

Next, let's implement `Game`. We'll put `#render` and `#update` there, but let's also add a new static method `::new` to encapsulate all the logic we'll use for instantiating our game object.

```rust
impl<'a> Game<'a> {
    pub fn new() -> Game<'a> {
        let bound = Bound {
            min: Point { x: 0, y: 0 },
            max: Point { x: 79, y: 49 }
        };
        let rc : Box<TcodRenderingComponent> = box RenderingComponent::new(bound);
        Game {
            exit: false,
            window_bounds: bound,
            rendering_component: rc
        }
    }

    pub fn render(&mut self, npcs: &Vec<Box<Updates>>, c: Character) {
        self.rendering_component.before_render_new_frame();
        for i in npcs.iter() {
            i.render(&mut self.rendering_component);
        }
        c.render(&mut self.rendering_component);
        self.rendering_component.after_render_new_frame();
    }

    pub fn update(&mut self, npcs: &mut Vec<Box<Updates>>, c: &mut Character, keypress: tcod::KeyState) {
        c.update(keypress, self);
        for i in npcs.mut_iter() {
            i.update(self);
        }
    }

    pub fn wait_for_keypress(&mut self) -> KeyState {
        self.rendering_component.wait_for_keypress()
    }
}
```

A couple things to take a look at. Our `#render` and `#update` methods now take a few less params. They don't need the game or rendering component because those are both now available internally. We had to implement `#wait_for_keypress` on Game because we call it from `main`, but `main` doesn't have the rendering component anymore. This is starting to feel like a code smell, but I'm going to leave it for now. Also note the way lifetime variables are defined on the struct and `::new` method.

Our main function is much more clean now:

```rust
fn main() {
    let mut game = Game::new();
    let mut c = Character::new(40, 25, '@');
    let d = box NPC::new(10, 10, 'd') as Box<Updates>;
    let ct = box NPC::new(40, 25, 'c') as Box<Updates>;
    let mut npcs: Vec<Box<Updates>> = vec![
        d, ct
    ];

    game.render(&npcs, c);
    while !(Console::window_closed() || game.exit) {
        // wait for user input
        let keypress = game.wait_for_keypress();

        // update game state
        match keypress.key {
            Special(key_code::Escape) => game.exit = true,
            _                         => {}
        }
        game.update(&mut npcs, &mut c, keypress);

        // render
        game.render(&npcs, c);
    }
}
```

It has a reference to the game, the npcs and the character, and that's about it. Actually, there's one quick simplification we can make:

```rust
let mut game = Game::new();
let mut c = Character::new(40, 25, '@');
let mut npcs: Vec<Box<Updates>> = vec![
    box NPC::new(10, 10, 'd') as Box<Updates>,
    box NPC::new(40, 25, 'c') as Box<Updates>
];
```

Every since we started using `Box<Updates>` for our NPCs, we don't actually need to initialize them outside the vector declaration.

Just like with other refactors, follow the compiler warnings / errors to help you prune dead code, make sure you get your types right and get rid of any unused imports. When you're done, take a look at our handiwork. We now have a completely decoupled rendering engine. We could swap out `libtcod` for anything other roguelike renderer and we'd be fine. The only place outside `TcodRenderComponent` that has a reference to tcod is in Game when we initialize the rendering component we're using.

### Input Component
The next place we're coupled is in the handling of input on our `Character` class. Let's split that out into a component. Instead of looking at separate input components, it makes more sense to have a `TcodInputMovementComopnent` to handle our characters input and a `RandomMovementComponent` for our dog and our cat. Let's make the `RandomMovementComponent` for our animals first.

We'll start by defining a new module, `movement` and a new trait, `MovementComponent`.

```rust
use util::Point;

pub trait MovementComponent {
    fn new() -> Self;
    fn update(&self, Point) -> Point;
}
```

This is a pretty tight contract. Our `#update` method on all of our components will only be allowed to take a point and return a point. We'll see how this works out for us. Let's implement the `RandomMovementComponent`. Unfortunately this isn't as easy as a cut and paste job. Because of the tight contract we made above, we need to solve a few problems. The first is that our `#update` method on `NPC` takes the `game` object as a parameter. Taking a look at that method, we passed in the `game` object because we needed to check the window bounds on a move. The easiest thing to do is to instantiate the movement component with the `game.window_bounds` so we have access to it.

```rust
use util::Point;

pub trait MovementComponent {
    fn new(Bound) -> Self;
    fn update(&self, Point) -> Point;
}
```

Then we can implement our `RandomMovementComonent` like this:

```rust
impl MovementComponent for RandomMovementComponent {
    fn new(bound: Bound) -> RandomMovementComponent {
        RandomMovementComponent { window_bounds: bound }
    }

    fn update(&self, point: Point) -> Point {
        let mut offset = Point { x: point.x, y: point.y };
        let offset_x = std::rand::task_rng().gen_range(0, 3i32) - 1;
        match self.window_bounds.contains(offset.offset_x(offset_x)) {
            DoesContain    => offset = offset.offset_x(offset_x),
            DoesNotContain => { return point; }
        }

        let offset_y = std::rand::task_rng().gen_range(0, 3i32) - 1;
        match self.window_bounds.contains(offset.offset_y(offset_y)) {
            DoesContain    => offset = offset.offset_y(offset_y),
            DoesNotContain => { return point; }
        }

        offset
    }
}
```

That's not so bad. We need to update how we instantiate our NPCs in `src/main.rs`:

```rust
let cmc : Box<RandomMovementComponent> = box MovementComponent::new(game.window_bounds);
let dmc : Box<RandomMovementComponent> = box MovementComponent::new(game.window_bounds);
let mut npcs: Vec<Box<Updates>> = vec![
    box NPC::new(10, 10, 'd', dmc) as Box<Updates>,
    box NPC::new(40, 25, 'c', cmc) as Box<Updates>
];
```

And lastly, we need to update our `NPC` struct and our `Updates` trait:

```rust
// NPC
fn update(&mut self) {
    self.position = self.movement_component.update(self.position);
}

// Updates
pub trait Updates {
    fn update(&mut self);
    fn render(&self, &mut RenderingComponent);
}
```

Turns out our `RandomMovementComponent` wasn't so bad to implement. Let's make one for our `Character`. We'll call this on `TcodUserMovementComponent`. This way if we were to ever move off of `tcod-rs` and use something like `ncurses`, we only have to instantiate a new movement component for our user. This one is a bit tougher. It requires the latest keypress from the console in order to figure out what to do, but we're not allowed to pass it as input. My solution, which isn't very rust-like, is to introduce a static method on `Game` that will return the last keypress recorded.

We define a global to track it and getter/setter methods for it:

```rust
static LAST_KEYPRESS : Option<KeyState> = None;

impl Game {
    pub fn get_last_keypress() -> Option<KeyState> {
        unsafe { LAST_KEYPRESS }
    }

    pub fn set_last_keypress(ks: KeyState) {
        unsafe { LAST_KEYPRESS = Some(ks); }
    }
}
```

We have to wrap accessing `LAST_KEYPRESS` in `unsafe` because `LAST_KEYPRESS` is a `static mut` (which is unsafe?). Next we want to update our `wait_for_keypress` method on `Game` to call `set_last_keypress` whenever we receive input.

```rust
pub fn wait_for_keypress(&mut self) -> KeyState {
    let ks = self.rendering_component.wait_for_keypress();
    Game::set_last_keypress(ks);
    return ks;
}
```

Now that we have a way to access the last keypress from anywhere, we can implement our new movement component:

```rust
pub struct TcodUserMovementComponent {
    window_bounds: Bound
}

impl MovementComponent for TcodUserMovementComponent {
    fn new(bound: Bound) -> TcodUserMovementComponent {
        TcodUserMovementComponent { window_bounds: bound }
    }

    fn update(&self, point: Point) -> Point {
        let mut offset = Point { x: point.x, y: point.y };
        offset = match Game::get_last_keypress() {
            Some(keypress) => {
                match keypress.key {
                    Special(key_code::Up) => {
                        offset.offset_y(-1)
                    },
                    Special(key_code::Down) => {
                        offset.offset_y(1)
                    },
                    Special(key_code::Left) => {
                        offset.offset_x(-1)
                    },
                    Special(key_code::Right) => {
                        offset.offset_x(1)
                    },
                    _ => { offset }
                }
            },
            None => { offset }
        };

        match self.window_bounds.contains(offset) {
            DoesContain    => { offset }
            DoesNotContain => { point }
        }
    }
}
```

As you can see the constructor is almost exactly the same, just uses a different return type. The `#update` method itself is really similar, but not quite. It has to do some extra juggling to unwrap `Game::get_last_keypress` (which might not be set yet). Notice that I also changed the `match` statement so that it returns a new point, which I assign to `offset`. That just seemed cleaner.

Now let's update our `Character` to use these newfound powers:

```rust
use util::Point;
use rendering::RenderingComponent;
use movement::MovementComponent;

pub struct Character<'a> {
    pub position:     Point,
    pub display_char: char,
    pub movement_component: Box<MovementComponent + 'a>
}

impl<'a> Character<'a> {
    pub fn new(x: i32, y: i32, dc: char, mc: Box<MovementComponent>) -> Character<'a> {
        Character { position: Point { x: x, y: y }, display_char: dc, movement_component: mc }
    }

    pub fn update(&mut self) {
        self.position = self.movement_component.update(self.position);
    }

    pub fn render(&self, rendering_component: &mut RenderingComponent) {
        rendering_component.render_object(self.position, self.display_char);
    }
}
```

And we'll update `src/main.rs` to pass in a new movement component to the character:

```rust
let char_mc : Box<TcodUserMovementComponent> = box MovementComponent::new(game.window_bounds);
let mut c = Character::new(40, 25, '@', char_mc);
```

An interesting thing happened when I tried to build the project at this point. I got an error about movement:

```
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:34:37: 34:38 error: use of moved value: `c`
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:34         game.update(&mut npcs, &mut c);
                                                                                                ^
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:24:24: 24:25 note: `c` moved here because it has type `dwemthys::character::Character`, which is non-copyable (perhaps you meant to use clone()?)
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:24     game.render(&npcs, c);
                                                                                   ^
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:37:28: 37:29 error: use of moved value: `c`
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:37         game.render(&npcs, c);
                                                                                       ^
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:24:24: 24:25 note: `c` moved here because it has type `dwemthys::character::Character`, which is non-copyable (perhaps you meant to use clone()?)
/Users/jmcfarland/code/rust/dwemthys-working/src/main.rs:24     game.render(&npcs, c);
```

I'm not really sure what's causing it, but changing it so that we passed a `&-ptr` instead of `c` itself solved this issue.

```rust
// src/main.rs
game.render(&npcs, &c);

// src/game/mod.rs
pub fn render(&mut self, npcs: &Vec<Box<Updates>>, c: &Character)
```

Cool! Now both our `NPC` and our `Character` are using a rendering component and a movement component. Actually... hold on a second... our `NPC` and our `Character` classes are *identical*. They both defer to their components for everything now. Let's merge them into a single class, `Actor`. To do this we'll change the `Character` class name to `Actor` and delete both `NPC` and `Updates` (we can actually remove the entire `traits` module). Here's what `src/main.rs` looks like now:

```rust
extern crate tcod;
extern crate dwemthys;

use dwemthys::game::Game;
use dwemthys::actor::Actor;
use dwemthys::rendering::RenderingComponent;
use dwemthys::movement::{MovementComponent, RandomMovementComponent, TcodUserMovementComponent};

use tcod::{Console, key_code, Special};

fn main() {
    let mut game = Game::new();
    let char_mc : Box<TcodUserMovementComponent> = box MovementComponent::new(game.window_bounds);
    let mut c = Actor::new(40, 25, '@', char_mc);
    let cmc : Box<RandomMovementComponent> = box MovementComponent::new(game.window_bounds);
    let dmc : Box<RandomMovementComponent> = box MovementComponent::new(game.window_bounds);
    let mut npcs: Vec<Box<Actor>> = vec![
        box Actor::new(10, 10, 'd', dmc),
        box Actor::new(40, 25, 'c', cmc)
    ];

    game.render(&npcs, &c);
    while !(Console::window_closed() || game.exit) {
        // wait for user input
        let keypress = game.wait_for_keypress();

        // update game state
        match keypress.key {
            Special(key_code::Escape) => game.exit = true,
            _                         => {}
        }
        game.update(&mut npcs, &mut c);

        // render
        game.render(&npcs, &c);
    }
}
```

We can actually clean that up a little bit more by implementing `Actor::dog`, `Actor::cat` and `Actor::heroine` methods:

```rust
pub fn dog(x: i32, y: i32, bound: Bound) -> Actor<'a> {
    let mc : Box<RandomMovementComponent> = box MovementComponent::new(bound);
    Actor::new(x, y, 'd', mc)
}

pub fn cat(x: i32, y: i32, bound: Bound) -> Actor<'a> {
    let mc : Box<RandomMovementComponent> = box MovementComponent::new(bound);
    Actor::new(x, y, 'c', mc)
}

pub fn heroine(x: i32, y: i32, bound: Bound) -> Actor<'a> {
    let mc : Box<TcodUserMovementComponent> = box MovementComponent::new(bound);
    Actor::new(x, y, '@', mc)
}
```

And changing `src/main.rs` to look like this

```rust
let mut game = Game::new();
let mut c = Actor::heroine(40, 25, game.window_bounds);
let mut npcs: Vec<Box<Actor>> = vec![
    box Actor::dog(10, 10, game.window_bounds),
    box Actor::cat(40, 25, game.window_bounds)
];
```

Cool! Look how clean that is. Make sure you run `cargo build` and clean up imports and dead code. I haven't been addressing that every time I move something around, but there's usually an associated import that needs to change. Speaking of building the project, go ahead and run `cargo run` and check out your handy work!

## Kobolds!
Okay. That was a lot of work, and I don't think we'll get to actual combat in this update, but we can at least put a monster on the screen. Let's render a kobold for our heroine to fight. We'll start off by defining a new constructor in our `Actor` class:

```rust
pub fn kobold(x: i32, y: i32, bound: Bound) -> Actor<'a> {
    let mc : Box<RandomMovementComponent> = box MovementComponent::new(bound);
    Actor::new(x, y, 'k', mc)
}
```

Then we'll add them to our NPCs array:

```rust
let mut npcs: Vec<Box<Actor>> = vec![
    box Actor::dog(10, 10, game.window_bounds),
    box Actor::cat(40, 25, game.window_bounds),
    box Actor::kobold(20, 20, game.window_bounds)
];
```

Now they should show up on the screen and move about randomly just like the rest of our NPCs. Check out how easy that was. Five lines of code to get a new character on the screen. Boo ya! But unfortunately we don't want a kobold that wanders around randomly. We want a kobold that ruthlessly pursues our heroine. Let's add a new movement component: `AggroMovementComponent`.

```rust
pub struct AggroMovementComponent {
    window_bounds: Bound
}

impl MovementComponent for AggroMovementComponent {
    fn new(bound: Bound) -> AggroMovementComponent {
        AggroMovementComponent { window_bounds: bound }
    }

    fn update(&self, point: Point) -> Point {
        point
    }
}
```

Notice I didn't fill in the `#update` method. That's because I hit a road block... in order to pursue the heroine this component needs to know where the heroine is. We can't pass the heroine's location in though. I'm going to continue to abuse globals until it bites me. Let's add a new one to track the location of our hero.

```rust
// src/game/mod.rs
static mut CHAR_LOCATION : Point = Point { x: 40, y: 25 };

impl Game {
    pub fn get_character_point() -> Point {
        unsafe { CHAR_LOCATION }
    }

    pub fn set_character_point(point: Point) {
        unsafe { CHAR_LOCATION = poin; }
    }
    
    pub fn update(&mut self, npcs: &mut Vec<Box<Actor>>, c: &mut Actor) {
        c.update();
        Game::set_character_point(c.position);
        for i in npcs.mut_iter() {
            i.update();
        }
    }
    
// src/actor/mod.rs
pub fn heroine(bound: Bound) -> Actor {
    let point = Game::get_character_point();
    let mc : Box<TcodUserMovementComponent> = box MovementComponent::new(bound);
    Actor::new(point.x, point.y, '@', mc)
}

// src/main.rs
let mut c = Actor::heroine(game.window_bounds);
```

That's not too bad of a change. Now we can implement our `AggroMovementComponent`:

```rust
fn update(&self, point: Point) -> Point {
    let char_point = Game::get_character_point();
    let mut offset = Point { x: 0, y: 0 };
    match point.compare_x(char_point) {
        RightOfPoint => offset = offset.offset_x(-1),
        LeftOfPoint  => offset = offset.offset_x(1),
        OnPointX     => {}
    }

    match point.compare_y(char_point) {
        BelowPoint => offset = offset.offset_y(-1),
        AbovePoint => offset = offset.offset_y(1),
        OnPointY   => {}
    }

    match point.offset(offset).compare(char_point) {
        PointsEqual    => { return point; },
        PointsNotEqual => {
            match self.window_bounds.contains(point.offset(offset)) {
                DoesContain    => { return point.offset(offset); }
                DoesNotContain => { return point; }
            }
        }
    }
}
```

First thing to notice is that I implemented some new compare methods on `Point`. The basic flow is, "check where the NPC is in relation to the character. If they're above, move up. If they're to the right, move right. Etc". Last thing it does is check whether or not we're in bounds, and whether or not our NPC is trying to move into the square the user occupies. If so, just cancel the move.

Our compare methods on `Point` look like this:

```rust
pub enum XPointRelation {
    LeftOfPoint,
    RightOfPoint,
    OnPointX
}

pub enum YPointRelation {
    AbovePoint,
    BelowPoint,
    OnPointY
}

pub enum PointEquality {
    PointsEqual,
    PointsNotEqual
}

impl Point {
    pub fn compare_x(&self, point: Point) -> XPointRelation {
        if self.x > point.x {
            RightOfPoint
        } else if self.x < point.x {
            LeftOfPoint
        } else {
            OnPointX
        }
    }

    pub fn compare_y(&self, point: Point) -> YPointRelation {
        if self.y > point.y {
            BelowPoint
        } else if self.y < point.y {
            AbovePoint
        } else {
            OnPointY
        }
    }

    pub fn compare(&self, point: Point) -> PointEquality {
        if self.x == point.x && self.y == point.y {
            PointsEqual
        } else {
            PointsNotEqual
        }
    }
}
```

Just like normal, use the compiler to help you prune dead code and get your imports right. After this you should be able to build your game and watch the kobold chase our heroine around. Nice!

Part three is getting pretty lengthy, so I don't think we'll make it to combat, but we've got a great base to build on now, so we'll get there in Part four for sure.

## Conclusion
We didn't get to adding in our combat engine, but our implementation of the Component Pattern will give us a lot of flexibility going forward. We also got to explore some really cool concepts in Rust. I think we're set up really well to figure out some of the more fun details in the game now.

## Tag
This post is [tagged version 3.1 over on GitHub](https://github.com/jaredonline/rust-roguelike/releases/tag/3.1).

## Next
[Part 4: Combat: Part II](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-4)

## Table of Contents
[Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents)

## Previous
[Part 2: Bringing Our Heroine to Life](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-2)