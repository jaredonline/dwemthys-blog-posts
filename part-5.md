<small style="font-size: 75%"><i>This is Part 5 in a many part series on how to make a roguelike game in Rust. If you're lost, check out the [Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents) to figure out where you should go.</i></small>

# Combat! Part III
Last time we made a ton of progress on combat. We've got a monster on the screen; we can issue combat instructions; we've got different parts of the screens reserved for different types of output. Things are looking pretty okay. Let's keep going!

## Refactoring
While I was working on [Part 4](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-4) I saw a guide go into pull request on the Rust GitHub. The guide was [The Rust Crates and Modules Guide](https://github.com/steveklabnik/rust/blob/module_guide/src/doc/guide-crates.md) and it was super duper exiting. I'd been really curious about how to do this The Right Way(tm) for awhile, and I was getting tired of this in my tab bar:

![mod mod mod](https://dl.dropboxusercontent.com/u/169446/dwemthys/part-5/mod.png)

It was getting confusing!

After reading the guide I learned that modules don't need to live in a folder (like `src/game/mod.rs`), they can just be named after the module! So I changed a lot of names, but didn't change any code. It was a pretty big refactor but it's *so good*. You can see the whole thing in [this commit](https://github.com/jaredonline/rust-roguelike/commit/3224528f0ee2b6d7380f0625628775fb457a0f4e). I suggest you take the time to go for it.

When I did this refactor I split rending into two separate modules, `renderers` and `windows`. That meant changing a bunch of my imports around. After I did that, I ran into this really weird error:

```
   Compiling dwemthys v0.0.1 (file:///Users/jmcfarland/code/rust/dwemthys)
/Users/jmcfarland/code/rust/dwemthys/src/main.rs:14:32: 14:61 error: source trait is private
/Users/jmcfarland/code/rust/dwemthys/src/main.rs:14     let mut b = Actor::heroine(game.windows.map.get_bounds());
                                                                                   ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

I spent some time Googling it but couldn't pin it down. I even spent some time on [play.rust-lang.org](http://play.rust-lang.org/) to try and create a repro to show the folks in the Mozilla #rust channel, but I couldn't reproduce it. So I added a function to `Windows`, called `#get_map_bounds`.

```rust
pub fn get_map_bounds(&self) -> Bound {
    self.map.get_bounds()
}
```

That got rid of the error. If anyone reading this knows what was up, feel free to send me an email, I'd love to know.

## Maps
One of the reasons we didn't get to implement combat in the last part was because we didn't have a good way to tell whether or not there was an enemy in the square the player was attacking. We actually didn't even have that great of a way to tell what square the player was attacking. We could use the players position to figure out which square they were attacking, but to find out if there was anything in that square we'd have to iterate our NPCs array and look for an actor in that position. That's not so great.

Implementing a map system should help us fix that. Maps are a pretty big change. A lot of code moved when I went through this exploration. The main motivator is to have a query interface to ask the `Maps` object whether or not there's an enemy on a square. In order to do that we'll move over all of the logic around iterating our vector of `Actor` objects to the `Maps` struct. This means that `GameState#update` will just call `Maps#update`. In order to preserve the order in which things are rendered we'll introduce the concept of "layers". Our `Maps` object will be made up of several attributes, and each of those will be a `Map`. Those are:

 - terrain
 - enemies
 - friends
 - PCs

 We haven't implemented terrain yet, but we will in a future tutorial.
 
Let's get started.

### Maps Struct
Let's start with the more simple of the two new structs we'll be introducing `Maps`.

```rust
pub struct Maps<'a> {
    pub terrain: Box<Map<'a>>,
    pub enemies: Box<Map<'a>>,
    pub friends: Box<Map<'a>>,
    pub pcs:     Box<Map<'a>>
}
```

`Maps` is very similar to our `Windows` struct, it's just a container for all the different maps we have. It will allow us to address a map by name if we need to, or all of them at once. We need the lifetime value `'a` because `Map` requires it. You'll see why in a minute.

The implementation of `Maps` is pretty straight forward:

```rust
impl<'a> Maps<'a> {
    pub fn new(size: Bound) -> Maps<'a> {
        let terrain = box Map::new(size);
        let enemies = box Map::new(size);
        let friends = box Map::new(size);
        let pcs     = box Map::new(size);

        Maps {
            friends: friends,
            enemies: enemies,
            terrain: terrain,
            pcs:     pcs
        }
    }

    pub fn update(&mut self, windows: &mut Windows) {
        self.pcs.update(windows);
        self.terrain.update(windows);
        self.friends.update(windows);
        self.enemies.update(windows);
    }

    pub fn render(&mut self, renderer: &mut Box<RenderingComponent>) {
        self.terrain.render(renderer);
        self.friends.render(renderer);
        self.enemies.render(renderer);
        self.pcs.render(renderer);
    }

    pub fn enemy_at(&self, point: Point) -> Option<&Box<Actor>> {
        let enemies_at_point = &self.enemies.content[point.x as uint][point.y as uint];
        if enemies_at_point.len() > 0 {
            Some(&enemies_at_point[0])
        } else {
            None
        }
    }
}
```

First, we have our `::new` method which you should be pretty familiar with by now. We pass it a single argument, `size: Bound` which we use to determine how big the map needs to be.

Our `#update` and `#render` methods are very simple, they just forward the command to each of the maps.

`#enemy_at` takes a single argument, the `point` and looks to see if the enemies map contains anything at that position. Right now it just returns a single enemy, assuming that the first enemy at that point is the one we want. We'll address that later. Notice that `#enemy_at` returns an `Option` instead of just the enemy. This way we can do pattern matching and properly program the "no enemy at location" scenario.

The implementation of `#enemy_at` clues you in a bit to how `Map` will be implemented. `Map` will store a nested set of `Vec` objects. It will have one for the x-axis, one for the y-axis and then one for all the actors in that location.

### Map Struct
So we have `Maps` all setup, we just need a `Map` object to store in it. Let's look at the struct:

```rust
pub struct Map<'a> {
    pub content: Vec<Vec<Vec<Box<Actor<'a>>>>>,
    pub size:    Bound
}
```

The `content` field stores all of our actors. If you think about it, you'll have a bunch of arrays like this when it's initialized:

```
[[[], [], []],
 [[], [], []],
 [[], [], []]]
```

Which loosely resemble the grid of our game. When we want to add an actor to `{1, 2}` we need to grab the second array, and then the first array inside that, ie `contents[2][1]`.

Let's go over the implementation:

```rust
impl<'a> Map<'a> {
    pub fn new(size: Bound) -> Map<'a> {
        let content = Map::init_contents(size);
        Map {
            content: content,
            size:    size
        }
    }

    pub fn init_contents(size: Bound) -> Vec<Vec<Vec<Box<Actor<'a>>>>> {
        let mut contents : Vec<Vec<Vec<Box<Actor>>>> = vec![];
        for _ in range(0, size.max.x) {
            let mut x_vec : Vec<Vec<Box<Actor>>> = vec![];
                for _ in range(0, size.max.y) {
                let y_vec : Vec<Box<Actor>> = vec![];
                x_vec.push(y_vec);
            }
            contents.push(x_vec);
        }
        return contents;
    }

    pub fn push_actor(&mut self, point: Point, actor: Box<Actor>) {
        self.content.get_mut(point.x as uint).get_mut(point.y as uint).push(actor);
    }

    pub fn update(&mut self, windows: &mut Windows) {
        let mut new_content = Map::init_contents(self.size);
        for x_iter in self.content.iter_mut() {
            for y_iter in x_iter.iter_mut() {
                for actor in y_iter.iter_mut() {
                    actor.update(windows);
                    if actor.is_pc {
                        Game::set_character_point(actor.position);
                    }
                    let point = actor.position;
                    let new_actor = actor.clone();
                    new_content.get_mut(point.x as uint).get_mut(point.y as uint).push(new_actor);
                }
            }
        }
        self.content = new_content;
    }

    pub fn render(&mut self, renderer: &mut Box<RenderingComponent>) {
        for (x, x_iter) in self.content.iter_mut().enumerate() {
            for (y, y_iter) in x_iter.iter_mut().enumerate() {
                for actor in y_iter.iter_mut() {
                    let point = Point::new(x as i32, y as i32);
                    renderer.render_object(point, actor.display_char);
                }
            }
        }
    }
}
```

First, let's go over `::new`. Notice it takes the same `size` param that `Maps::new` took. It makes a call out to `init_contents` to create the nested `Vec`s and then it creates and returns the `Map` object. `::new` takes a lifetime parameter `'a` and uses it for creating the `Box<Actor<'a>>`.

We use `::init_contents` to create the nested vector. Originally I did this because it was fairly messy logic and I wanted to contain it in its own method, but it came in handy later on. It's fairly straightforward though. It iterates a given number of times in nested loops to create the appropriately sized arrays. This is so that we can access `contents[1][2]` without having to first check and see if it exists.

`#push_actor` is a helper method to add an actor to the appropriate `Vec`. I didn't want to expose the implementation details of `Maps` so this seemed like the best choice.

`#update` is where we'll put the logic to update all the actors on the map. Before we just iterated our NPC array and then called update on the heroine. Now we're delegating that to `Maps` and `Map`.  It loops through it's nested structure and calls update on each actor. Notice that it doesn't stop there. It does some weird stuff.

First it checks `is_pc` on the `Actor` (a field we haven't added yet) and if so it updates the `CHARACTER_POINT` constant in `Game`. That's because we removed the concept of the character from the main game loop.

Next it calls `Actor#clone` on the actor. Another method that doesn't exist yet. It uses that to create a new `Actor` object and then it stores it in the new vector we create at the beginning of the method. After doing this for all the actors, it replaces `Map#content` with `new_content`. We do this to avoid complicated ownership issues when trying to update `content` directly in the for loops. Because we borrow `self` at the beginning of the loops we can't modify `self` from within the for loops.

Next we have `#render`. `#render` takes in a `RenderingComponent` and uses that to draw the characters on screen based on their location. Notice that it uses the location from the map and not the one stored on the character itself. We use the map as the authoritative source of location information. The location stored on the `Actor` is just for convenience. (This will probably bite us in the ass later.)

You'll have to import a bunch of stuff to get this to compile, but you can use the compiler to figure that part out [=

### Actor
Let's go over the changes to actor mentioned above. First we need the `is_pc` field:

```rust
pub struct Actor<'a> {
    pub id:           uint,
    pub position:     Point,
    pub display_char: char,
    pub movement_component: Box<MovementComponent + 'a>,
    pub is_pc: bool
}
```

Pretty simple actually, just add a new `bool`. Next we need to modify all our constructors (`::heroine`, `::cat`, `::dog`, `::kobold`) to use this field. All of them should be `false` except `::heroine`. I'll leave that as an exercise for you.

Next we need to implement the `#clone` method. Rust is pretty smart about some things. If you call `#clone` on an object it assumes you mean to use the `core::clone::Clone` trait, so when we go to implement `Actor#clone` we'll do it by implementing the `Clone` trait:

```rust
impl<'a> Clone for Actor<'a> {
    fn clone(&self) -> Actor<'a> {
        let mc = self.movement_component.box_clone();
        Actor::new(self.position.x, self.position.y, self.display_char, mc, self.is_pc)
    }
}
```

It's pretty easy. We just create a new `Actor` with all the same fields as the current one. The only weird thing is calling `#box_clone` on `movement_component`. I couldn't get the regular `Clone` trait there to work so I asked in the Rust IRC channel and they helped me out. Apparently you can't define a method that has both `&self` as a variable and `Self` as a return type.

That's it for `Actor`, let's talk about `MovementComponent` next.

### MovementComponent
The only real change to `MovementComponent` we need to make is to add the `#box_clone` method. First we'll add its signature to the `MovementComponent` trait:

```rust
pub trait MovementComponent {
    fn new(Bound) -> Self;
    fn update(&self, Point, &mut Windows) -> Point;
    fn box_clone(&self) -> Box<MovementComponent>;
}
```

Then we'll add the implementation to each of our movement components. Here it is for `UserMovementComponent`:

```rust
fn box_clone(&self) -> Box<MovementComponent> {
    box UserMovementComponent { window_bounds: self.window_bounds }
}
```

I'll leave the rest of the implementations up to you. (Maybe we'll turn this into a macro someday?)

Once we've done this we'll have made all the changes we need to get `Maps` and `Map` to work. Next we need to integrate them!

### Game
First thing to do is add `Maps` as a field to `Game`:

```rust
pub struct Game<'a, 'b> {
    pub exit:                bool,
    pub window_bounds:       Bound,
    pub rendering_component: Box<RenderingComponent + 'a>,
    pub game_state:          Box<GameState          + 'a>,
    pub windows:             Windows<'a>,
    pub maps:                Maps<'b>
}
```

Notice that we also added a new lifetime parameter to `Game`. That's because both `Windows` and `Maps` need their own. I'm not sure why `Windows` and the `Box`'s can share though.

We had to update the method signature of `Game::new` to match (as well as the `imple` statement for `Game`).

```rust
pub fn new() -> Game<'a, 'b> {
    // init all the bounds

    // init our windows and rendering component

    // init our game state

    let maps = Maps::new(map_bounds);

    Game {
        exit:                false,
        window_bounds:       total_bounds,
        rendering_component: rc,
        windows:             windows,
        game_state:          gs,
        maps:                maps
    }
}
```

Sweet! Now we have a `maps` field on `Game`. Let's start changing some shit! First thing we'll do is change `Game#render` and `GameState#render`. They'll no longer take a vector of npcs or the character as arguments. `Game#render` turns into:

```rust
pub fn render(&mut self) {
    self.game_state.render(&mut self.rendering_component, &mut self.maps, &mut self.windows);
}
```

And `GameState#render` turns into:

```rust
fn render(&mut self, renderer: &mut Box<RenderingComponent>, maps: &mut Maps, windows: &mut Windows) {
    renderer.before_render_new_frame();
    let mut all_windows = windows.all_windows();
    for window in all_windows.iter_mut() {
        renderer.attach_window(*window);
    }
    maps.render(renderer);
    renderer.after_render_new_frame();
}
```

We got rid of the iteration of the `npcs` object and the call to `character.render` and just call `#render` on `maps` instead.

Next we'll do the same thing to `Game#update` and `GameState#update`. He's `Game#update`:

```rust
pub fn update(&'a mut self) {
    if self.game_state.should_update_state() {
        self.game_state.exit();
        self.update_state();
        self.game_state.enter(&mut self.windows);
    }

    self.game_state.update(&mut self.maps, &mut self.windows);
}
```

Same story. Get rid of `npcs` and `character` and just use `maps`. `GameState#update` becomes:

```rust
fn update(&mut self, maps: &mut Maps, windows: &mut Windows);
```

We change the method signature to reflect what we're passing the individual components. Here's how that plays out in `MovementGameState#update`:

```rust
fn update(&mut self, maps: &mut Maps, windows: &mut Windows) {
    match Game::get_last_keypress() {
        Some(ks) => {
            match ks.key {
                // Because Shift is used for attack keys we don't want to do
                // anything when it's pushed. We can check for shift when we
                // process the next keypress
                SpecialKey(KeyCode::Shift) => {},
                _ => {
                    maps.update(windows);
                }
            }
        },
        _    => {}
    }
}
```

If we get a movement key we just call `#update` on `maps`. The change to `AttackInputGameState#update` is a bit more involved:

```rust
fn update(&mut self, maps: &mut Maps, windows: &mut Windows) {
    match Game::get_last_keypress() {
        Some(ks) => {
            let mut msg = "You attack ".to_string();
            let mut point = Game::get_character_point();
            match ks.key {
                SpecialKey(KeyCode::Up) => {
                    point = point.offset_y(-1);
                    msg.push_str("up");
                    self.should_update_state = true;
                },
                SpecialKey(KeyCode::Down) => {
                    point = point.offset_y(1);
                    msg.push_str("down");
                    self.should_update_state = true;
                },
                SpecialKey(KeyCode::Left) => {
                    point = point.offset_x(-1);
                    msg.push_str("left");
                    self.should_update_state = true;
                },
                SpecialKey(KeyCode::Right) => {
                    point = point.offset_x(1);
                    msg.push_str("right");
                    self.should_update_state = true;
                },
                _ => {}
            }

            if self.should_update_state {
                match maps.enemy_at(point) {
                    Some(_) => {
                        msg.push_str(" with your ");
                        msg.push_str(self.weapon.as_slice());
                        msg.push_str("!");
                        windows.messages.buffer_message(msg.as_slice());
                    },
                    None => {
                        windows.messages.buffer_message("No enemy in that direction!");
                    }
                }
            }
        },
        _ => {}
    }
}
```

First we updated the methods signature to match the new signature. Then we update the `match` statement to track a point. The point tracked represents where the user is attacking. Then we call `#enemy_at` with the point being tracked to see if there's an enemy at in that square. If there is we use the same string building logic used before. If not we buffer a different message that says there's no enemy in that square. Cool!

### Main
Since we've changed `Game` and how we're storing our actors, we need to update our main game loop.

First we'll update how we're creating and registering our actors:

```rust
let mut game = Game::new();
game.maps.friends.push_actor(Point::new(10, 10), box Actor::dog(10, 10, game.windows.get_map_bounds()));
game.maps.friends.push_actor(Point::new(40, 25), box Actor::cat(40, 25, game.windows.get_map_bounds()));
game.maps.enemies.push_actor(Point::new(20, 20), box Actor::kobold(20, 20, game.windows.get_map_bounds()));
game.maps.pcs.push_actor(Game::get_character_point(), box Actor::heroine(game.windows.get_map_bounds()));
```

This is super janky but it works for now.

Next we need to change all of our calls to `Game#render` and `Game#update` to not use any argument:

```rust
game.render()
while !(Console::window_closed() || game.exit) {
  // game loop
  
  game.update();
  game.render();
}
```

### Movement Component
Now we need to implement our `#box_clone` method for `MovementComponent` and all the different components we have.

First we'll add the method signature to our trait:

```rust
fn box_clone(&self) -> Box<MovementComponent>;
```

Then we'll add an implementation to our components:

```rust
fn box_clone(&self) -> Box<MovementComponent> {
    box AggroMovementComponent { window_bounds: self.window_bounds }
}
```

I'll let you add the rest of the methods.

That's that! We now have a map system for our game.

## Removing Globals
One of the coolest parts about writing this tutorial is getting to share cool new things with lots of people. Because I'm learning Rust just fast enough to implement the next part of the tutorial, I don't always do things The Rust Way(tm). Fortunately awesome people reading this reach out to me and explain a better way to do it. Recently GitHub use [GBGamer](https://github.com/GBGamer) sent me an email explaining a way to not use globals and linked me to [their version of the game](https://github.com/GBGamer/roguelike-rs). Instead of using global vars they suggested using the `RefCell` object in conjunction with the `Rc` object.

I hadn't heard of either of these, so I looked it up. My understanding (please correct me if you're reading this and I'm wrong) is that the `Rc` object is a reference-counted, shared pointer and it's immutable. Instead of Rust deciding at compile time when something will be freed, it's decided at run time. As soon as the last reference to it is gone, it's freed. You can read more about it at the [Rust Rc Docs](http://doc.rust-lang.org/std/rc/).

The `RefCell` object is a type that is allowed to be mutated through shared pointers (`&T` vs `&mut T`). `RefCell` requires you to get a write-lock on it before writing to it. This is done by calling `#borrow` on it. You can read more about `RefCell` (and it's sibling `Cell`) on [the Rust docs](http://doc.rust-lang.org/std/cell/).

So what does that mean for us? It means we can take all of our information that was global before, put it in a struct, put that struct in a `RefCell` and that `RefCell` into an `Rc` and pass around copies of it to anything that will need it. Let's see how that's done.

### MoveInfo
First we'll create a new struct called `MoveInfo` which will hold all of our global movement based information:

```rust
pub struct MoveInfo {
    pub last_keypress: Option<KeyboardInput>,
    pub char_location: Point,
    pub bounds: Bound
}

impl MoveInfo {
    pub fn new(bound: Bound) -> MoveInfo {
        MoveInfo {
            last_keypress: None,
            char_location: Point::new(40, 25),
            bounds: bound
        }
    }
}
```

It's very straightforward. It takes all the information we used to store globally and moves it into the struct. I moved the map bounds into the struct as well because it was accessed in so many places.

### Using MoveInfo
This is a pretty far-reaching change. First off, any file that's using `Rc` and `RefCell` need these two import lines:

```rust
use std::cell::RefCell;
use std::rc::Rc;
```

Second, we need to add a `move_info` as an attribute on `Game`:

```rust
pub struct Game<'a, 'b> {
    // stuff
    move_info: Rc<RefCell<MoveInfo>>
}
```

Then we need to update our `Game::new` method to create the `MoveInfo` object:

```rust
pub fn new() -> Game<'a, 'b> {
    // create the bounds and states

    let move_info = Rc::new(RefCell::new(MoveInfo::new(map_bounds)));
    let maps = Maps::new(move_info.clone());

    Game {
        exit:                false,
        window_bounds:       total_bounds,
        rendering_component: rc,
        windows:             windows,
        game_state:          gs,
        maps:                maps,
        move_info:           move_info
    }
}
```

Notice we're passing `move_info.clone()` to `Maps::new` instead of the `map_bounds` we were passing before. Keep in mind that `move_info` is not a `MoveInfo` object but a `Rc<RefCell<MoveInfo>>` object, so when call `#clone` on it, we're calling `Rc#clone`, not `MoveInfo#clone`. We're not cloning the data structure, but the reference-counted reference to it. We'll do this every time we pass `move_info` to something.

Next we need to find all instances of `Game::set_last_keypress`, `Game::get_last_keypress`, `Game::get_character_point` and `Game::set_character_point` and replace them with calls into `MoveInfo`. This also means passing `MoveInfo` to all movement components, game states, actors and maps. You can see my commit that introduced this change [here](https://github.com/jaredonline/rust-roguelike/commit/3fe64f01950f7168123e182fa6d5b3c9b861a06e). I won't walk through each one because it's pretty repetitive, but I will call out a couple things.

#### Scoping

The first is "scoping" (is that the right word?). Access the fields in `move_info` is not as simple as:

```rust
move_info.last_keypress
```

That's because `Rc` doesn't have a field named `last_keypress`. Instead we have to jump through a number of hoops. First we have to call `#borrow` on the `Rc`. This gives us an immutable pointer to the `RefCell`, which we then have to dereference with `#deref` which gets us a pointer to the `MoveInfo` object.

```rust
move_info.borrow().deref().last_keypress
```

The unfortunate part is that we need a way to tell the compiler when to release the "borrow" (I kind of think of them as locks). If you attempt to borrow something that is already borrowed your program will crash and burn with an error like this:

```rust
task '<main>' failed at 'RefCell<T> already borrowed', /build/rust-git/src/rust/src/libcore/cell.rs:306
```

That's a runtime failure, and not a very descriptive one.  How do we keep this from happening? We create new scopes for all of our borrows, and when the scopes end the borrow is released:

```rust
{
  move_info.borrow().deref().last_keypress
}
```

If we need to store a value from `MoveInfo` in a variable we do so like this:

```rust
let last_keypress = {
  move_info.borrow().deref().last_keypress
};
```

Then we can reference `last_keypress` later in the method. Setting values is fairly straight forward, but you need to use the `_mut()` versions of the methods:

```rust
{
	move_info.borrow_mut().deref_mut().last_keypress = keypress;
}
```

#### Initialization
The last thing I want to mention is the game initialization in `src/main.rs`, specifically the `Actor` initialization. It's not too complicated, but can cause headaches. Here's mine:

```rust
game.maps.friends.push_actor(Point::new(10, 10), box Actor::dog(10, 10, game.move_info.clone()));
game.maps.friends.push_actor(Point::new(40, 25), box Actor::cat(40, 25, game.move_info.clone()));
game.maps.enemies.push_actor(Point::new(20, 20), box Actor::kobold(20, 20, game.move_info.clone()));

let char_location = {
    game.move_info.borrow().deref().char_location
};
game.maps.pcs.push_actor(char_location, box Actor::heroine(game.move_info.clone()));
```

The big thing to note is that we no longer pass around the bounds (which has a cascading effect). We pass around the `Rc<RefCell<MoveInfo>>` which means we need to call `#clone` on it when passing it to new objects that will be storing the reference.

#### Finishing up
You should be able to use my commit and the notes above to figure out how to implement the rest of this refactor. At the end you get to do my favorite thing, delete some code. Get rid of the constants `LAST_KEYPRESS` and `CHAR_LOCATION` in `src/game.rs` and the static methods on `Game`.

## Lifetimes
About three weeks passed between when I started this article and when I finished it. That's a long time in Rust land. When I came back to the project, it didn't compile anymore. I was having a lot of lifetime issues. I made a series of changes which mostly removed named lifetimes and replaced them with `'static` lifetimes. You can see the sum changes in [this commit range](https://github.com/jaredonline/rust-roguelike/compare/791b92af1e171160722ce02ce94861ed60142be3...fba4e28b314d595d3e2579af63c2c24428f669ed).

In that time tcod-rs also underwent some breaking changes, so in that commit range you'll see some related changes.

## Conclusion
I think this is a good place to stop for this part. We just took a huge step towards getting combat working. Next time we'll look at dealing and receiving damage. Here's the [GitHub tag for v5.0](https://github.com/jaredonline/rust-roguelike/releases/tag/5.0).

## Table of Contents
[Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents)

## Previous
[Part 4: Combat! Part II](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-4)
