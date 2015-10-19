<small style="font-size: 75%"><i>This is Part 4 in a many part series on how to make a roguelike game in Rust. If you're lost, check out the [Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents) to figure out where you should go.</i></small>

# Combat! Part II
Last post was a bit of a doozy. I'm really hoping we actually get to implement some combat this time. Thinking through what we need for combat I came up with the following rough list:
 
 * input characters for attack (`/` for sword, `^` for boomerang, `*` for bomb and `%` for lettuce)
 * the ability to halt the normal input -> update -> render cycle
 * the ability to take additional input (ie, "Which direction are you using your sword?")
 * the ability to prompt the user and display messages

That's a lot of requirements. Fortunately, we don't have any refactoring to start with in this part. The first three in that list are really closely related, so I'm gonna leave them for later. Let's do the last bullet first, "The ability to prompt the user and display messages".

To do that, we'll need a part of the screen that is reserved for rendering messages and prompts. Right now we restrict actor movement to within the window bounds, but we'll actually have to create a sub-section in our window that will hold the game map. While we're doing this, let's think about any other sections we might want:
 
 * one for player input
 * one for character stats
 * one for player notifications

Let's reserve the right hand side of our game for character stats, the very bottom for our player notifications and above that we'll have a section for prompts. Let's say we want our stats section to be 20 characters wide and as tall as our map. We'll make our notifications section 10 characters tall and as wide as our window, and our input section 2 characters tall and as wide as our window. The rest of the screen will be filled with the map (which we should also put in a section).

## Windows
`libtcod` has this concept of windows or consoles. They can be nested inside each other by jumping through a bunch of hoops. The nice thing is they can be cleared and rendered separate of one another. We'll use them for our different sub-sections. To do this we'll want a new type of component, a `WindowComponent` that we can give to our `RenderingComponent` to handle rendering.

I'm not sure this is the best approach, but I created a different type of window component for each of our windows. I did all of this in `src/rendering/mod.rs`:

```rust
struct TcodStatsWindowComponent;
struct TcodInputWindowComponent;
struct TcodMessagesWindowComponent;
struct TcodMapWindowComponent;
```

The idea here is that if we want different window behavior we change from a `Map` to a `Messages` type, and if we want a new rendering engine we'd change from `Tcod` to something else. We'll see if that works out.

Let's take a look at our base trait `WindowComponent`:

```rust
pub trait WindowComponent {
    fn new(Bound) -> Self;

    fn get_bounds(&self)      -> Bound;
    fn get_bg_color(&self)    -> Color;
    fn get_console(&mut self) -> &mut Console;

    fn clear(&mut self) {
        let color       = self.get_bg_color();
        let mut console = self.get_console();
        console.set_default_background(color);
        console.clear();
    }

    fn print_message(&mut self, x: int, y: int, alignment: tcod::TextAlignment, text: &str) {
        let mut console = self.get_console();
        console.print_ex(x, y, background_flag::Set, alignment, text);
    }
}
```

Notice that `#get_bg_color` returns an instance of `Color`? We'll have to import that module in our `src/rendering/mod.rs` file along with the rest of our Tcod imports:

```rust
use self::tcod::{Console, background_flag, KeyState, Color};
```

`WindowComponent` defines two default methods, `#clear` and `#print_message`. These are the same for every implementation right now, so we define them here.  In order for them to work, they can't actually access attributes on the structs that will inherit them (the compiler isn't smart enough to figure out if a struct that implements `WindowComponent` has the right attributes). That's why they define the `#get_` methods.

Here's a sample implementation of the structs (this one is `TcodStatsWindowComponent`):

```rust
pub struct TcodStatsWindowComponent {
    pub console:          Console,
    pub background_color: Color,
    bounds:               Bound
}

impl WindowComponent for TcodStatsWindowComponent {
    fn new(bounds: Bound) -> TcodStatsWindowComponent {
        let height = bounds.max.y - bounds.min.y + 1;
        let width  = bounds.max.x - bounds.min.x + 1;
        let mut console = Console::new(
            width  as int,
            height as int,
        );

        let red = Color::new(255u8, 0u8, 0u8);
        TcodStatsWindowComponent {
            console:          console,
            background_color: red,
            bounds:           bounds
        }
    }

    fn get_console(&mut self) -> &mut Console { &mut self.console     }
    fn get_bounds(&self)      ->      Bound   { self.bounds           }
    fn get_bg_color(&self)    ->      Color   { self.background_color }
}
```

It's pretty straightforward. They each define a constructor (which builds its own `tcod::Console`), background color and bounds. The bounds are used to determine both this sub-windows size and position. I won't show you here, but the other windows have the exact same patterns, but they use different background colors (I did this so I could make sure I was rendering the windows in the right place while testing).

Let's take a look at the initialization code:

```rust
pub fn new() -> Game {
    let total_bounds   = Bound::new(0,  0, 99, 61);
    let stats_bounds   = Bound::new(79, 0, 99, 49);
    let input_bounds   = Bound::new(0, 50, 99, 52);
    let message_bounds = Bound::new(0, 53, 99, 61);
    let map_bounds     = Bound::new(0,  0, 78, 49);

    let rc  : Box<TcodRenderingComponent>      = box RenderingComponent::new(total_bounds);
    let sw  : Box<TcodStatsWindowComponent>    = box WindowComponent::new(stats_bounds);
    let iw  : Box<TcodInputWindowComponent>    = box WindowComponent::new(input_bounds);
    let mw  : Box<TcodMessagesWindowComponent> = box WindowComponent::new(message_bounds);
    let maw : Box<TcodMapWindowComponent>      = box WindowComponent::new(map_bounds);

    Game {
        exit:                false,
        window_bounds:       total_bounds,
        rendering_component: rc,
        stats_window:        sw,
        input_window:        iw,
        messages_window:     mw,
        map_window:          maw
    }
}
```

In our `Game::new()` method we're now initializing all of our window components alongside our rendering component. We also added some new attributes to our `Game` struct. My thinking is that at some point we'll want to call methods on these different windows (perhaps something like `render_character_stats(&mut actor)`) based on input or what's happening in the game loop. We'll see.

Also notice I added a `::new()` method to bound as a shortcut. I was tired of typing the same thing over and over again. Here's the implementation:

```rust
pub fn new(min_x: i32, min_y: i32, max_x: i32, max_y: i32) -> Bound {
    Bound {
        min: Point { x: min_x, y: min_y },
        max: Point { x: max_x, y: max_y }
    }
}
```

If you try to build the game now, make sure you add the implementations for the other three window components, otherwise you'll get loads of errors. Copy/pasting the existing implementations will work for now, just make sure you change the names. Here's a [link to mine on GitHub](https://github.com/jaredonline/rust-roguelike/blob/7ae0c76b480dd7ec3ccfcf238ea977bbb3b96562/src/rendering/mod.rs) (there's probably some changes between yours and mine, but you should be able to get the gist).

Now we need to have a way to attach these windows to the main console. We'll add a new method to our `RenderingComponent` trait to handle this code because this code is highly coupled to the rendering library we're using. In our `impl RenderingComponent for TcodRenderingComponent {` block we'll add:

```rust
fn attach_window(&mut self, window: &mut Box<WindowComponent>) {
    window.clear();
    window.print_message(0, 0, tcod::Left, "Sup foo!");
    window.print_message(0, 1, tcod::Left, "Nothin fool!");
    let bounds  = window.get_bounds();
    let console = window.get_console();

    Console::blit(&*console, 0, 0, (bounds.max.x as int) + 1, (bounds.max.y as int) + 1, &mut self.console, bounds.min.x as int, bounds.min.y as int, 1f32, 1f32);
}
```

It took me a lot of testing and digging into the libtcod and tcod-rs source to figure out how ot make this work. The libtcod documentation [talks about blitting](http://doryen.eptalys.net/data/libtcod/doc/1.5.1/html2/console_offscreen.html#6), but not in a lot of detail. The gist is you need to take one console, and apply it on top of another. You have specify the alpha-transparency of the foreground and background of the console being "blitted" (layered on top of the root console). So here, we call `Console::blit` and say we want to blit from `0,0` to the edge of our window. We'll use the window bounds to figure out where to put it, and we want our window to be 100% opaque for the foreground and background.

Now that we have all that in place, we can wire these up so they actually render. To do this we only have to modify `Game#render` to look like this:

```rust
pub fn render(&mut self, npcs: &Vec<Box<Actor>>, c: &Actor) {
    self.rendering_component.before_render_new_frame();
    self.rendering_component.attach_window(&mut self.stats_window);
    self.rendering_component.attach_window(&mut self.input_window);
    self.rendering_component.attach_window(&mut self.messages_window);
    self.rendering_component.attach_window(&mut self.map_window);
    for i in npcs.iter() {
        i.render(self.rendering_component);
    }
    c.render(self.rendering_component);
    self.rendering_component.after_render_new_frame();
}
```

It's a bit repetitive, but I couldn't figure out a clean way to do this with arrays in Rust and not mess up mutability and ownership. If you `cargo run` you should see:

![Windows!](https://dl.dropboxusercontent.com/u/169446/dwemthys/part-4/windows.png)

But there's a problem. You can move your character outside of the main map window. Launch the game and just hold the right arrow button, and our heroine will march all the way into the red stats window. That's no good. The fix is pretty simple. Right now we initialize our `MovementComponent` with a `Bound`. It happens that we do it with the `Game#window_bounds` at the moment, but it should be straight forward to move that to the map bounds.

Let's just change that here in `src/main.rs`

```rust
let mut game = Game::new();
let mut c = Actor::heroine(game.map_window.get_bounds());
let mut npcs: Vec<Box<Actor>> = vec![
    box Actor::dog(10, 10, game.map_window.get_bounds()),
    box Actor::cat(40, 25, game.map_window.get_bounds()),
    box Actor::kobold(20, 20, game.map_window.get_bounds())
];
```

Build and run the game and now all the characters will stay inside the map. Cool!

## Combat!
Let's get back to the idea of combat. Here was our to-do list to make combat work:

 * input characters for attack (`/` for sword, `^` for boomerang, `*` for bomb and `%` for lettuce)
 * the ability to halt the normal input -> update -> render cycle
 * the ability to take additional input (ie, "Which direction are you using your sword?")
 * the ability to prompt the user and display messages

We kind of did the last one. We can make windows... and one of those windows can be for prompting the user for input and the other for displaying messages. But we don't actually have a good way of doing that right now, but we're close. What we need now is a way to buffer messages into a window and then print them every time we render the window.

First we need a `Vec` to keep track of the messages, and we'll want a way to know how many messages each window is capable of displaying. We'll add that to each of our structs that implement `WindowComponent`:

```rust
pub struct TcodInputWindowComponent {
    // other traits
    messages:             Vec<Box<String>>,
    max_messages:         uint
}
```

We'll need to add new getters for both of these traits to each component as well:

```rust
fn get_mut_messages(&mut self) -> &mut Vec<Box<String>> { &mut self.messages    }
fn get_messages(&self)         ->      Vec<Box<String>> { self.messages.clone() }
fn get_max_messages(&self)     ->      uint             { self.max_messages     }
```

Notice how there's two ways to get the messages. One returns the a mutable pointer to the vector and the other returns a clone of the vector. You'll see why in a minute.

Next we need a way to push messages onto our `Vec`. In our `WindowComponent` trait we'll add:

```rust
fn buffer_message(&mut self, text: &str) {
    let max      = self.get_max_messages();
    let message  = String::from_str(text);
    let messages = self.get_mut_messages();

    messages.insert(0, box message);
    messages.truncate(max);
}
```

Here we use the mutable version of `get_messages`, `get_mut_messages`. We need a mutable version so that we can push values on to it. 

Next we'll change our `attach_window` method in `TcodRenderComponent` so that it renders our messages right before attaching the window.

```rust
fn attach_window(&mut self, window: &mut Box<WindowComponent>) {
    window.clear();
    let mut line = 0i;
    let bounds   = window.get_bounds();
    let messages = window.get_messages();

    for message in messages.iter() {
        window.print_message(0, line, tcod::Left, message.as_slice());
        line = line + 1;
    }

    let console  = window.get_console();

    Console::blit(&*console, 0, 0, (bounds.max.x as int) + 1, (bounds.max.y as int) + 1, &mut self.console, bounds.min.x as int, bounds.min.y as int, 1f32, 1f32);
}
```

Here we use the immutable version of `get_messages`. That's because if we use the mutable version we borrow `window` too many times as a mutable pointer. We can only do that once per scope.

Right now we're just printing all the messages, one after the other, each on a new line. Nothing fancy, but it'll work.

Lastly, let's implement a method on `WindowComponent` that will let us flush the messages easily:

```rust
fn flush_buffer(&mut self) {
    let max      = self.get_max_messages();
    let messages = self.get_mut_messages();

    for _ in range(0, max) {
        messages.insert(0, box String::from_str(""));
    }
    messages.truncate(max);
}
```

This one is pretty straight forward. Just loop over our messages and replace them with empty strings. If you haven't seen the `_` syntax in Rust before, it's how you call methods with "placeholder" variables you don't intend to use.

Let's add some code to `Game` to test our handiwork. Let's change the `wait_for_keypress` method to look like this:

```rust
pub fn wait_for_keypress(&mut self) -> KeyState {
    let ks = self.rendering_component.wait_for_keypress();
    match ks.key {
        Printable('/') => self.input_window.buffer_message("Wich direction would you like to attack with your heoric sword? [Press an arrow key]"),
        _              => self.input_window.flush_buffer()
    }

    Game::set_last_keypress(ks);
    return ks;
}
```

Now if you press "/" in game it'll prompt you to tell us which direction you want to swing. Neat! At this point I changed the background color for all of my windows to black (`Color::new(0u8, 0u8, 0u8)`) so it looked better. Here's what it should look like:

![Prompting the user](https://dl.dropboxusercontent.com/u/169446/dwemthys/part-4/prompt.png)

I also did a big refactor to make my life easier. I started using macros for all the shared code in the different window components. Macros in Rust are a bit weird, here's how I got mine to work. First, I added this compiler flag to the top of `src/lib.rs`: 

```rust
#![feature(macro_rules)]
```

Next I added the following three macros:

```rust
macro_rules! window_component_getters(
    () => {
        fn get_console(&mut self)      -> &mut Console          { &mut self.console     }
        fn get_bounds(&self)           ->      Bound            { self.bounds           }
        fn get_bg_color(&self)         ->      Color            { self.background_color }
        fn get_mut_messages(&mut self) -> &mut Vec<Box<String>> { &mut self.messages    }
        fn get_max_messages(&self)     ->      uint             { self.max_messages     }
        fn get_messages(&self)         ->      Vec<Box<String>> { self.messages.clone() }
    }
)

macro_rules! window_component_def(
    ($name:ident) => {
        pub struct $name {
            pub console:          Console,
            pub background_color: Color,
            bounds:               Bound,
            messages:             Vec<Box<String>>,
            max_messages:         uint
        }
    }
)

macro_rules! window_component_init(
    ($name:ident, $color:expr, $max_messages:expr) => {
        fn new(bounds: Bound) -> $name {
            let height = bounds.max.y - bounds.min.y + 1;
            let width  = bounds.max.x - bounds.min.x + 1;
            let console = Console::new(
                width  as int,
                height as int,
            );

            $name {
                console:          console,
                background_color: $color,
                bounds:           bounds,
                messages:         vec![],
                max_messages:     $max_messages
            }
        }
    }
)
```

Macros in rust are defined using the `macro_rules!` command, followed by a name. After the name, wrapped in `()` you have the macro body. It starts with `() =>` and is followed by expressions between `{}`. It's a bit convoluted and I don't fully understand it, so I'd read the [Rust Macros Guide](http://doc.rust-lang.org/guide-macros.html) for more information.

Afterwards, all of my `WindowComponent` definitions look like this:

```rust
window_component_def!(TcodStatsWindowComponent)
impl WindowComponent for TcodStatsWindowComponent {
    window_component_init!(TcodStatsWindowComponent, Color::new(0u8, 0u8, 0u8), 10u)
    window_component_getters!()
}

window_component_def!(TcodInputWindowComponent)
impl WindowComponent for TcodInputWindowComponent {
    window_component_init!(TcodInputWindowComponent, Color::new(0u8, 0u8, 0u8), 2u)
    window_component_getters!()
}

window_component_def!(TcodMapWindowComponent)
impl WindowComponent for TcodMapWindowComponent {
    window_component_init!(TcodMapWindowComponent, Color::new(0u8, 0u8, 0u8), 10u)
    window_component_getters!()
}

window_component_def!(TcodMessagesWindowComponent)
impl WindowComponent for TcodMessagesWindowComponent {
    window_component_init!(TcodMessagesWindowComponent, Color::new(0u8, 0u8, 0u8), 10u)
    window_component_getters!()
}
```

And that makes handling things *a lot* easier.

Alright, let's take a look at our todo list again:

 * input characters for attack (`/` for sword, `^` for boomerang, `*` for bomb and `%` for lettuce)
 * the ability to halt the normal input -> update -> render cycle
 * the ability to take additional input (ie, "Which direction are you using your sword?")
 * ~~the ability to prompt the user and display messages~~

Let's take a crack at the first three all at once, because they're all related.

## State Machine
In order to process attack input separate from movement input, we need to be able to put the game into different modes. Every time the game goes through it's loop, we need to make different decisions based on the games mode. This is a pretty classic fit for the [State Machine Pattern](http://gameprogrammingpatterns.com/state.html). This pattern let's us put the game into different modes, and we'll have an object that represents each mode. Instead of processing the game loop inside of `Game` directly, we'll delegate to the active mode object (which we'll call a state). Sounds pretty straight forward... let's see how hard it is to implement.

First thing to do is replace our current logic with our base state. We'll need a new trait, `GameState`, a new struct, `MovementGameState` and an implementation for that trait.

Our trait will look like this:

```rust
pub trait GameState {
    fn new() -> Self;

    fn update(&mut self, npcs: &mut Vec<Box<Actor>>, character: &mut Actor);
    fn render(&mut self, renderer: &mut Box<RenderingComponent>, npcs: &Vec<Box<Actor>>, character: &Actor, windows: &mut Vec<&mut Box<WindowComponent>>);
}
```

The `::new` method you're familiar with by now. `#update` has the same method signature as it does on `Game`, but `#render` has some additional components. It needs to know which rendering component we're going to render, and it needs a list of windows to render.

The struct doesn't have any state (yet):

```rust
pub struct MovementGameState;
```

And the implementation is fairly straight forward:

```rust
impl GameState for MovementGameState {
    fn new() -> MovementGameState {
        MovementGameState
    }

    fn update(&mut self, npcs: &mut Vec<Box<Actor>>, character: &mut Actor) {
        character.update();
        Game::set_character_point(character.position);
        for npc in npcs.iter_mut() {
            npc.update();
        }
    }

    fn render(&mut self, renderer: &mut Box<RenderingComponent>, npcs: &Vec<Box<Actor>>, character: &Actor, windows: &mut Vec<&mut Box<WindowComponent>>) {
        renderer.before_render_new_frame();
        for window in windows.iter_mut() {
            renderer.attach_window(*window);
        }
        for npc in npcs.iter() {
            npc.render(renderer);
        }
        character.render(renderer);
        renderer.after_render_new_frame();
    }
}
```

One neat thing we've done in `#render` is iterate over an array of windows instead of having four calls to the same method typed out. That DRY's things up a little. Next we'll need to update `#render` and `#update` in `Game`:

```rust
pub fn render(&mut self, npcs: &Vec<Box<Actor>>, c: &Actor) {
    let mut windows = vec![
        &mut self.stats_window,
        &mut self.input_window,
        &mut self.messages_window,
        &mut self.map_window
    ];
    self.game_state.render(&mut self.rendering_component, npcs, c, &mut windows);
}

pub fn update(&mut self, npcs: &mut Vec<Box<Actor>>, c: &mut Actor) {
    self.game_state.update(npcs, c);
}
```

That is pretty straight forward. Notice the `mut Vec` we make in `#render` as a list of all the windows. Lastly, we need to add the `game_state` to our `Game` struct definition and initialize a new state when we initialize our game:

```rust
pub struct Game<'a> {
    pub exit:                bool,
    pub window_bounds:       Bound,
    pub rendering_component: Box<RenderingComponent + 'a>,
    pub stats_window:        Box<WindowComponent    + 'a>,
    pub map_window:          Box<WindowComponent    + 'a>,
    pub input_window:        Box<WindowComponent    + 'a>,
    pub messages_window:     Box<WindowComponent    + 'a>,
    pub game_state:          Box<GameState          + 'a>
}

impl<'a> Game<'a> {
    fn new() -> Game<'a> {
		  // a bunch of stuff
		  
		  let gs : Box<MovementGameState> = box GameState::new();

        Game {
            exit:                false,
            window_bounds:       total_bounds,
            rendering_component: rc,
            stats_window:        sw,
            input_window:        iw,
            messages_window:     mw,
            map_window:          maw,
            game_state:          gs
        }
	}
}
```

Now you should be able to build and run your game without any issues. Next we need an input state and a way to switch between the two. Each state will be responsible for deciding when it's done, and then it will delegate to the `Game` object to figure out which state to go to.

The easiest way to accomplish this, for now, is to add some code to `Game#update`. On every update call, we'll ask the state if it needs to be updated, and if it does, we'll figure out which state we should be in.

```rust
pub fn update(&'a mut self, npcs: &mut Vec<Box<Actor>>, c: &mut Actor) {
    if self.game_state.should_update_state() {
        self.game_state.exit();
        self.update_state();
        self.game_state.enter(&mut self.windows);
    }

    self.game_state.update(npcs, c, &mut self.windows);
}
```

Notice we added two new methods to `GameState`, `#enter` and `#exit`. `#enter` will be called every time we enter to a new state, and `#exit` every time we're done with a state. This gives us the chance to initialize or break down anything a state needs. We'll add default implementations to the `GameState` trait (we'll also add the method signature for `#should_update_state`):

```rust
pub trait GameState {
    fn enter(&self, &mut Windows) {}
    fn exit(&self)  {}

    fn should_update_state(&self) -> bool;
}
```

Next we need to add a `#should_update_state` method to our `MovementGameState` implementation:

```rust
fn should_update_state(&self) -> bool {
    true
}
```

The easiest thing to do is have the movement state (which is the default state) *always* tell the game that it's done. This way, at the end of every input command, we'll check to see if we need to switch states.

## A new Window struct
At this point I realized that the `AttackInputGameState` was going to need to access not just an array of `Box<WindowComponent>`'s, but individual window components, and by name. So I quickly refactored some code. First, I created a new struct, `Windows`:

```rust
pub struct Windows<'a> {
    pub stats:    Box<WindowComponent + 'a>,
    pub map:      Box<WindowComponent + 'a>,
    pub input:    Box<WindowComponent + 'a>,
    pub messages: Box<WindowComponent + 'a>
}
```

I added an implementation that would give me an array of all the windows for the times I needed them:

```rust
impl<'a > Windows<'a > {
    fn all_windows(&'a mut self) -> Vec<&mut Box<WindowComponent>> {
        let windows = vec![
            &mut self.stats,
            &mut self.input,
            &mut self.messages,
            &mut self.map
        ];

        return windows;
    }
}
```

And changed the `Game` struct to:

```rust
pub struct Game<'a> {
    pub exit:                bool,
    pub window_bounds:       Bound,
    pub rendering_component: Box<RenderingComponent + 'a>,
    pub game_state:          Box<GameState          + 'a>,
    pub windows:             Windows<'a>
}
```

This required lots of other refactors, like changing `Game::new()` to:

```rust
pub fn new() -> Game<'a> {
    let total_bounds   = Bound::new(0,  0, 99, 61);
    let stats_bounds   = Bound::new(79, 0, 99, 49);
    let input_bounds   = Bound::new(0, 50, 99, 52);
    let message_bounds = Bound::new(0, 53, 99, 61);
    let map_bounds     = Bound::new(0,  0, 78, 49);

    let rc  : Box<TcodRenderingComponent>      = box RenderingComponent::new(total_bounds);

    let sw  : Box<TcodStatsWindowComponent>    = box WindowComponent::new(stats_bounds);
    let iw  : Box<TcodInputWindowComponent>    = box WindowComponent::new(input_bounds);
    let mw  : Box<TcodMessagesWindowComponent> = box WindowComponent::new(message_bounds);
    let maw : Box<TcodMapWindowComponent>      = box WindowComponent::new(map_bounds);

    let windows = Windows {
        input:    iw,
        messages: mw,
        map:      maw,
        stats:    sw
    };

    let gs : Box<MovementGameState> = box GameState::new();

    Game {
        exit:                false,
        window_bounds:       total_bounds,
        rendering_component: rc,
        windows:             windows,
        game_state:          gs
    }
}
```

I basically just followed the compiler warnings/errors to find all the places I needed to change how I was passing around the windows. This included changing the `GameState` trait to:

```rust
pub trait GameState {
    fn new() -> Self;

    fn enter(&self, &mut Windows) {}
    fn exit(&self)  {}

    fn update(&mut self, npcs: &mut Vec<Box<Actor>>, character: &mut Actor, windows: &mut Windows);
    fn render(&mut self, renderer: &mut Box<RenderingComponent>, npcs: &Vec<Box<Actor>>, character: &Actor, windows: &mut Windows);

    fn should_update_state(&self) -> bool;
}
```

And adjusting the `MovementGameState` to match. This was a very, very handy refactoring.

## Back to our regularly scheduled program
Now that we've got the `Windows` struct handy, let's add our `AttackInputGameState` which we'll switch to when our player says they want to attack. First, the struct definition:

```rust
pub struct AttackInputGameState {
    should_update_state: bool,
    weapon: String
}
```

Notice this struct has two fields (where our `MovementGameState` struct had none). The `should_update_state` bool will help us keep track of whether or not we need to update, and the `weapon` string is just a bit of a hack I used to say which weapon we're attacking with.

Now we can implement the `GameState` trait for `AttackInputGameState`:

```rust
impl GameState for AttackInputGameState {
    fn new() -> AttackInputGameState {
        AttackInputGameState {
            should_update_state: false,
            weapon: "".to_string()
        }
    }

    fn should_update_state(&self) -> bool {
        self.should_update_state
    }

    fn enter(&self, windows: &mut Windows) {
        windows.input.flush_buffer();
        let mut msg = "Which direction do you want to attack with ".to_string();
        msg.push_str(self.weapon.as_slice());
        msg.push_str("? [Use the arrow keys to answer]");
        windows.input.buffer_message(msg.as_slice())
    }

    fn update(&mut self, _: &mut Vec<Box<Actor>>, _: &mut Actor, windows: &mut Windows) {
        match Game::get_last_keypress() {
            Some(ks) => {
                let mut msg = "You attack ".to_string();
                match ks.key {
                    Special(key_code::Up) => {
                        msg.push_str("up");
                        self.should_update_state = true;
                    },
                    Special(key_code::Down) => {
                        msg.push_str("down");
                        self.should_update_state = true;
                    },
                    Special(key_code::Left) => {
                        msg.push_str("left");
                        self.should_update_state = true;
                    },
                    Special(key_code::Right) => {
                        msg.push_str("right");
                        self.should_update_state = true;
                    },
                    _ => {}
                }

                if self.should_update_state {
                    msg.push_str(" with your ");
                    msg.push_str(self.weapon.as_slice());
                    msg.push_str("!");
                    windows.messages.buffer_message(msg.as_slice());
                }
            },
            _ => {}
        }
    }

    fn render(&mut self, renderer: &mut Box<RenderingComponent>, npcs: &Vec<Box<Actor>>, character: &Actor, windows: &mut Windows) {
        renderer.before_render_new_frame();
        let mut all_windows = windows.all_windows();
        for window in all_windows.iter_mut() {
            renderer.attach_window(*window);
        }
        for npc in npcs.iter() {
            npc.render(renderer);
        }
        character.render(renderer);
        renderer.after_render_new_frame();
    }
}
```

In the `::new` method we set the `should_update_state` flag to a default of `false` and the weapon to an empty string. In `#should_update_state` we actually check our flag instead of always returning true. This is because we don't want to switch states until the user has told us which direction to attack.

In `#enter` we clear the message buffer in our input window component and add a new message which asks the user where they want to attack.

In `#update` we take a look at the last key press, looking for a directional key. If one was hit we set our flag `should_update_state` to true, and push a message to the messages window component to confirm the users action.

In `#render` we do the same thing as in the `MovementGameState`, so I just made this the default `GameState` implementation (and removed it from the other game states):

```rust
pub trait GameState {
    fn new() -> Self;

    fn enter(&self, &mut Windows) {}
    fn exit(&self)  {}

    fn update(&mut self, npcs: &mut Vec<Box<Actor>>, character: &mut Actor, windows: &mut Windows);
    fn render(&mut self, renderer: &mut Box<RenderingComponent>, npcs: &Vec<Box<Actor>>, character: &Actor, windows: &mut Windows) {
        renderer.before_render_new_frame();
        let mut all_windows = windows.all_windows();
        for window in all_windows.iter_mut() {
            renderer.attach_window(*window);
        }
        for npc in npcs.iter() {
            npc.render(renderer);
        }
        character.render(renderer);
        renderer.after_render_new_frame();
    }

    fn should_update_state(&self) -> bool;
}
```

The last thing we need to do is add a method to game, `Game#update_state` which will decide which state to use. This method is a bit long, but I'll explain why after:

```rust
fn update_state(&mut self) {
    match Game::get_last_keypress() {
        Some(ks) => {
            match ks.key {
                Printable('/') => {
                    let mut is : Box<AttackInputGameState> = box GameState::new();
                    is.weapon = "Heroic Sword".to_string();
                    self.game_state = is as Box<GameState>;
                },
                Special(key_code::Number6) => {
                    if ks.shift {
                        let mut is : Box<AttackInputGameState> = box GameState::new();
                        is.weapon = "Boomerang".to_string();
                        self.game_state = is as Box<GameState>;
                    }
                },
                Special(key_code::Number8) => {
                    if ks.shift {
                        let mut is : Box<AttackInputGameState> = box GameState::new();
                        is.weapon = "Deadly Bomb".to_string();
                        self.game_state = is as Box<GameState>;
                    }
                },
                Special(key_code::Number5) => {
                    if ks.shift {
                        let mut is : Box<AttackInputGameState> = box GameState::new();
                        is.weapon = "Delicious Lettuce".to_string();
                        self.game_state = is as Box<GameState>;
                    }
                },
                Special(key_code::Shift) => {}
                _ => {
                    let ms : Box<MovementGameState> = box GameState::new();
                    self.game_state = ms as Box<GameState>;
                }
            }
        },
        _ => {}
    }
}
```

This method grabs the last keypress and decides which state we should be in. The default is to enter the movement state. If we push the printable key `/` we enter the attack state. But for the others, we have to do some craziness. First we have to check to see if we're in the special key that corresponds to number each weapon is on. For example, the `%` key is on the same key as `5`, so we check to see if we pressed `5`. If we did, we then check to see if the shift key was on. If it was, then we switch to the attack input state for that weapon. A bit janky, but it works.

At this point, it should compile and play. Make sure to follow any compiler warnings about missing imports. Let's take a look at our to-do list:

 * ~~input characters for attack (`/` for sword, `^` for boomerang, `*` for bomb and `%` for lettuce)~~
 * ~~the ability to halt the normal input -> update -> render cycle~~
 * ~~the ability to take additional input (ie, "Which direction are you using your sword?")~~
 * ~~the ability to prompt the user and display messages~~

We've successfully crossed them all off! But unfortunately, we still can't kill that kobold. We've got some cleanup to do, then we'll address the kobold.

## Leaky Abstraction
You may have noticed something. All of a sudden our states and our `Game` object know about tcod specific code. Specifically, to get the code to render in the last section we had to add:

```rust
use self::tcod::{key_code, Special, Printable};
```

Now if we want to change our implementation library, we'll need to do more than code up new components. We'll need to fix `Game` as well. I'm not a huge fan of this. Let's make a new component, a `TcodInputComponent`. This will be used only for taking input from the user, translating it to a neutral key state, and storing it.

First let's define our version of `KeyState`, which we'll call `KeyboardInput`:

```rust
pub struct KeyboardInput {
    key: Key
}
```

Fairly straight-forward. `tcod-rs` has a lot of fields on `KeyState`, but so far we've only been using `key`, so we'll just keep that for now.

Next, we'll add our versions of `Printable`, `Special`, and `key_code`:

```rust
pub type KeyCode = self::KeyCode::KeyCode;
pub mod KeyCode {
    pub enum KeyCode {
        // Arrow keys
        Up,
        Down,
        Left,
        Right,

        // Special
        Shift,
        Escape,

        // Default
        None
    }
}
```

Again, ours are very basic. We don't need all of the functionality that `tcod-rs` has, so we'll only port over a subset. To get the module to import the way I wanted I had to do a bit of juggling with the module names. To avoid a compiler warning, I added `#![allow(non_snake_case)]` to the top of this module. I got a lot of help on this part from [this GitHub ticket](https://github.com/rust-lang/rust/issues/10090).

Next, we'll define the trait, `InputComponent`:

```rust
pub trait InputComponent<T> {
    fn new() -> Self;
    fn translate_input(&self, T) -> KeyboardInput;
}
```

Finally, we'll implement our trait for our struct:

```rust
pub struct TcodInputComponent;

impl InputComponent<KeyState> for TcodInputComponent {
    fn new() -> TcodInputComponent { TcodInputComponent }

    fn translate_input(&self, key_state: KeyState) -> KeyboardInput {
        let key : Key = if key_state.shift {
            match key_state.key {
                self::tcod::Special(key_code::Number5) => Printable('%'),
                self::tcod::Special(key_code::Number6) => Printable('^'),
                self::tcod::Special(key_code::Number8) => Printable('*'),
                _                                      => SpecialKey(KeyCode::None)
            }
        } else {
            match key_state.key {
                self::tcod::Printable('/')            => Printable('/'),
                self::tcod::Special(key_code::Up)     => SpecialKey(KeyCode::Up),
                self::tcod::Special(key_code::Down)   => SpecialKey(KeyCode::Down),
                self::tcod::Special(key_code::Left)   => SpecialKey(KeyCode::Left),
                self::tcod::Special(key_code::Right)  => SpecialKey(KeyCode::Right),
                self::tcod::Special(key_code::Shift)  => SpecialKey(KeyCode::Shift),
                self::tcod::Special(key_code::Escape) => SpecialKey(KeyCode::Escape),
                _                                     => SpecialKey(KeyCode::None)
            }
        };

        KeyboardInput { key: key }
    }
}
```

The implementation is pretty straight forward. We just do some pattern matching on the input, "wash" it of it's Tcod-ness and return it. One nice thing we get from doing this is the ability to define our weapon keys as their own `Printable`s.

After doing this we can clean up any reference to Tcod in `Game`. Let's look at `wait_for_keypress` first. Here's the new implementation: 

```rust
pub fn wait_for_keypress(&mut self) -> KeyboardInput {
    let key_state = self.rendering_component.wait_for_keypress();

    Game::set_last_keypress(key_state);
    return key_state;
}
```

As you can see, I've really buried the change down in `game.rendering_component`. So let's take a look there. First we'll want to instantiate and store a new `InputComponent` on the `RenderingComponent`:

```rust
pub struct TcodRenderingComponent<'a> {
    pub console: Console,
    pub input_component: Box<InputComponent<KeyState> + 'a>
}

// in the TcodRenderingComponent impl
fn new(bounds: Bound) -> TcodRenderingComponent<'a> {
    let console = Console::init_root(
        (bounds.max.x + 1) as int,
        (bounds.max.y + 1) as int,
        "libtcod Rust tutorial", false
    );

    let ic : Box<TcodInputComponent> = box InputComponent::new();

    TcodRenderingComponent {
        console: console,
        input_component: ic
    }
}
```

That's fairly straightforward, we've seen it plenty of times. Next let's take a look at `wait_for_keypress` here:

```rust
fn wait_for_keypress(&self) -> KeyboardInput {
  let ks = self.console.wait_for_keypress(true);
  self.input_component.translate_input(ks)
}
```

That's also pretty straightforward. Instead of just returning whatever `console` returns, we now wash it through our input component first, and return that.

Returning to `Game`, we can clean also clean up `#update_state`:

```rust
fn update_state(&mut self) {
    match Game::get_last_keypress() {
        Some(ks) => {
            match ks.key {
                Printable('/') => {
                    let mut is : Box<AttackInputGameState> = box GameState::new();
                    is.weapon = "Heroic Sword".to_string();
                    self.game_state = is as Box<GameState>;
                },
                Printable('^') => {
                    let mut is : Box<AttackInputGameState> = box GameState::new();
                    is.weapon = "Boomerang".to_string();
                    self.game_state = is as Box<GameState>;
                },
                Printable('*') => {
                    let mut is : Box<AttackInputGameState> = box GameState::new();
                    is.weapon = "Deadly Bomb".to_string();
                    self.game_state = is as Box<GameState>;
                },
                Printable('%') => {
                    let mut is : Box<AttackInputGameState> = box GameState::new();
                    is.weapon = "Delicious Lettuce".to_string();
                    self.game_state = is as Box<GameState>;
                },
                _ => {
                    let ms : Box<MovementGameState> = box GameState::new();
                    self.game_state = ms as Box<GameState>;
                }
            }
        },
        _ => {}
    }
}
```

There's another module that deals with user input we need to clean up. And this is a really cool side-effect I hadn't anticipated. We no longer need `TcodUserMovementComponent`, we can just convert it to `UserMovementComponent`. That's because the movement component, when it calls `Game::get_last_keypress`, will get a non-Tcod keypress. Here's what the new implementation looks like:

```rust
impl MovementComponent for UserMovementComponent {
    fn update(&self, point: Point, windows: &mut Windows) -> Point {
        let mut offset = Point { x: point.x, y: point.y };
        offset = match Game::get_last_keypress() {
            Some(keypress) => {
                match keypress.key {
                    SpecialKey(KeyCode::Up) => {
                        offset.offset_y(-1)
                    },
                    SpecialKey(KeyCode::Down) => {
                        offset.offset_y(1)
                    },
                    SpecialKey(KeyCode::Left) => {
                        offset.offset_x(-1)
                    },
                    SpecialKey(KeyCode::Right) => {
                        offset.offset_x(1)
                    },
                    _ => { offset }
                }
            },
            None => { offset }
        };

        match self.window_bounds.contains(offset) {
            DoesContain    => {
                offset
            }
            DoesNotContain => {
                windows.messages.buffer_message("You can't move that way!");
                point
            }
        }
    }
}
```

Notice I took the opportunity to buffer a new message in the `UserMovementComponent`. If the user tries to move off the enge of the window, we buffer a new message telling them they can't go that way. Kinda dorky, but now our game feels more interactive.

!["You can't move that way!"](https://dl.dropboxusercontent.com/u/169446/dwemthys/part-4/cantgothatway.png)

## Next time...
Given the length of this post, I don't think we'll get to combat this time around. Right now our player can tell the heroine to attack, and in what direction, but we have no way of knowing if there's an enemy there or not. We also don't have any means for calculating damage, storing hit points, or telling the kobold how to attack back.

Next time around we'll try to address this with a Map system and a Combat system.

## Conclusion
Holy crap! I feel like we got through a lot in that one. We now have window areas, input and output, and game states! The GitHub tag for this is [v4.0](https://github.com/jaredonline/rust-roguelike/releases/tag/4.0).

## Next
[Part 5: Combat! Part III](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-5)

## Table of Contents
[Table of Contents](http://jaredonline.svbtle.com/roguelike-tutorial-table-of-contents)

## Previous
[Part 3: Combat!](http://jaredonline.svbtle.com/roguelike-tutorial-in-rust-part-3)