The reason you hit a wall with Rust and window managers is that you're trying to use a declarative learning loop on a procedural problem.

When you configure Hyprland (Lua), Neovim (Lua), or Qtile (Python), the architecture already exists. The state machine, event loop, window tree, and X11/Wayland bindings are pre-built by someone else. You are supplying a hash map of options, callbacks, and keybind tables. The application calls _into_ your script.

Writing a window manager in Rust flips that model completely upside down. You aren't configuring someone else's state machine—**you are writing the state machine**. Rust doesn't use configuration files for system programs; it uses code, domain-specific crates, or embedded scripting (like Lua or TOML) exposed via bindings.

If your proven learning method is **"tear apart a working, highly configurable target, mess with its internals, and build mental models through direct iteration,"** you don't need a book or a video. You need the right Rust targets that expose their architecture through hackable code bases or Rust-native configuration.

Here are the exact footholds to anchor your methodology in Rust.

### 1. The Direct Equivalent: Configurable Rust Software

If you learn best by configuring real programs written in Rust, start with tools where the user configuration _is_ Rust code or tightly bound Rust structs:

- **LeftWM (`leftwm`)**: A tiling window manager written in Rust for X11. It uses a modular architecture (split into a core daemon, worker, and state components). While themeing uses Liquid templates, its IPC and internal state architecture are extremely clean and modular.
- **AWC / Custom Rust WMs**: Look at `**penrose**`—a modular window manager library for X11 written in Rust. **Penrose is literally designed to be configured by writing a custom Rust binary.** You don't write a `.conf` file; you write a `main.rs` that imports `penrose`, configures the workspace layout logic, hooks up keybindings, and compiles it. This is the exact Rust equivalent to Qtile or xmonad (Haskell).
- **Wayland Compositors in Rust**: Look at `**niri**` (a scrollable-tiling Wayland compositor written in Rust) or `**cardboard**`. _Niri_ in particular is modern, cleanly structured Rust using `smithay`.

### 2. The Bridge: Building a WM via `penrose` or `smithay`

Instead of trying to raw-dog XCB bindings or Wayland protocols from scratch (which leads to the video/book brain-melt you're experiencing), leverage Rust's compositor and WM framing libraries.

#### For X11: Use `penrose`

`penrose` acts like a toolkit. It handles the X11 connection, event loop, and protocol plumbing so you can focus purely on layout logic, state management, and keybindings in Rust.

1. Clone the `penrose` repository or set up a new Cargo binary crate (`cargo new my-wm`).
2. Add `penrose` as a dependency in `Cargo.toml`.
3. Your configuration file _is_ `src/main.rs`:

use penrose::{
    core::{bindings::KeyEventHandler, Config, WindowManager},
    x11rb::RustConn,
    Result,
};

fn main() -> Result<()> {
    // 1. Define your hooks, keybindings, and layouts here in pure Rust
    let config = Config::default();
    let conn = RustConn::new()?;
    
    // 2. Instantiate the WM
    let mut wm = WindowManager::new(config, conn, vec![], default_hooks())?;
    
    // 3. Run the event loop
    wm.run()
}

By hacking on a `penrose` project, you get the exact "tinker with config → see immediate screen reaction" feedback loop you used for Qtile and Lua.

#### For Wayland: Study `smithay`

If you are aiming for Wayland (like Hyprland or Sway), nobody writes raw C/Wayland protocol bindings from scratch in Rust unless they are building the framework itself. They use **Smithay**.

- Smithay is a modular library for building Wayland compositors in Rust (the Rust equivalent to C's `wlroots`).
- Don't read the docs top-to-bottom. Clone the `smithay` repo and go directly to `anvil/` (their reference implementation compositor). Run `anvil`, break the layout code in `src/`, recompile (`cargo run`), and see what happens.

### 3. How to Shift Your Mental Model for Rust

In Lua or Python, objects are fluid and state can be mutated anywhere. Rust will throw a wall of borrow-checker errors if you try to structure a window manager using naive object-oriented patterns (e.g., a `Window` holding a pointer to a `Workspace`, which holds a list of `Window`s).

When inspecting these Rust WM codebases, look for these three patterns—they are the backbone of all Rust WMs:

1. **The Event Loop (`calloop` or `mio`)**: The WM spends 99% of its time asleep waiting for an event (keypress, mouse move, new window mapped). When an event comes in, it matches on an `enum`.
2. **State Vectors / Tree Indexing**: Instead of complex pointer webs, Rust WMs typically store windows in a `Vec<WindowId>` or an arena tree (`indextree` / `petgraph`), managing focus using indices.
3. **IPC via Sockets**: Modern Rust WMs split the core logic from the client config by reading JSON/Unix sockets (similar to `hyprctl` or `i3-msg`).

### Next Action Step

Stop watching videos. Set up a local crate using `penrose` for X11 or clone `smithay`'s `anvil` for Wayland. Start treating the Rust codebase as your config file, change the layout algorithms or keybinding handlers directly in source, and let `cargo check` teach you the language rules as you hack.

First off: **Niri existing does not make your project pointless.**

Niri is a scrollable tiling compositor written for a specific, opinionated workflow. Hyprland is a feature-packed dynamic tiling compositor with complex animations, custom render pipelines, window rules, and heavy IPC. Bringing Hyprland’s density, aesthetic control, and feature set into a smooth-scrolling or custom layout engine built on Smithay is a massive, incredibly cool project.

The most important shift right now is realizing **you do not need to drop down to Penrose/X11 to get a foothold.** If your end target is Wayland with Smithay, starting in X11 with Penrose will actually confuse you because Wayland and X11 handle rendering, display servers, and input devices completely differently.

Since you learn by taking a working target, tweaking its code, and breaking things to see what happens, here is how you launch into **Smithay** directly using your existing workflow.

### The Real Foothold: Smithay’s `anvil`

Smithay is not a single executable—it is a toolkit of Rust crates. However, the Smithay repository contains a fully working reference compositor called `**anvil**`.

`anvil` is to Smithay what a default `config.py` is to Qtile. It is a minimal, working Wayland compositor that uses Smithay under the hood for everything: input handling, DRM/KMS, Wayland socket management, and rendering.

#### How to start hacking on `anvil`:

1. **Clone the Smithay repo into your dev environment:**
    
    git clone https://github.com/Smithay/smithay.git
    cd smithay
    
2. **Run Anvil directly in nested mode (inside your current Wayland session):**
    
    cargo run --bin anvil -- --winit
    
    _This spawns a window inside your current desktop running a sub-Wayland compositor._
3. **Break and rewrite its internals:** Instead of reading Smithay docs top-to-bottom, open `anvil/src/` in your editor and look at how it handles the exact things you want to customize:
    - `**state.rs**`: This is where the core compositor state lives (the equivalent of your global config/WM state).
    - `**input_handler.rs**`: Look at how keybindings and mouse actions are intercepted and dispatched.
    - **`drawing.rs` or `render.rs`**: Look at how surfaces and textures are rendered to the screen.

### How to Build "Niri + Hyprland" Step-by-Step

Trying to build the whole vision at once will lead back to feeling stuck. Break it down into hackable Rust milestones inside `anvil`:

#### Phase 1: Modify Keybindings and Window Spawning

Go into Anvil’s input handling logic. Strip out their default keybinds and replace them with your own. Add custom commands that log to `stdout` or spawn terminal windows (`ghostty`, `foot`, etc.) when pressed. This gets you comfortable with Rust’s `match` statements and enum-based event handling.

#### Phase 2: Hack the Layout Algorithm

Anvil has basic window tracking. Look at how window geometry (`Rectangle`) is calculated when a new Wayland surface is mapped.

- Replace their default floating/tiling placement with a basic 1D horizontal stack (the primitive form of Niri's infinite scroll).
- Calculate `x` offsets based on window index: `window_x = index * window_width`.

#### Phase 3: Add Custom Surface Offsets (The "Niri" Infinite Strip)

Instead of keeping windows locked to screen coordinates, introduce a global `viewport_offset_x` variable in your state.

- Render windows relative to `(window_x - viewport_offset_x)`.
- Wire your scroll wheel or keybindings to modify `viewport_offset_x`. Suddenly, you have a scrolling compositor prototype.

#### Phase 4: Layer on Hyprland Features

Once the layout engine moves, you start pulling in Hyprland concepts:

- **Window Rules**: Create Rust structs that match on Wayland app-IDs/titles and override window properties on launch.
- **Animations**: Animate `viewport_offset_x` using a spring/bezier curve crate (like `keyframe` or custom lerp math) inside the frame render loop.
- **IPC**: Add a Unix domain socket listener that parses incoming JSON commands (your own `hyprctl`).

### The Mindset Shift for Smithay

In Hyprland/Lua or Qtile/Python, you write callbacks like `on_window_open(win)`.

In Smithay, you implement traits. Smithay fires events into a central loop (`calloop`), and your compositor struct implements Smithay's event handlers:

// Simplified example of how Smithay hands you a new window:
impl CompositorHandler for MyCompositor {
    fn new_surface(&mut self, surface: WlSurface) {
        // You decide where to put it in your window list/tree
        self.windows.push(surface);
        self.recalculate_layout();
    }
}

By working inside `anvil`, all the low-level graphics setup is already running. Your job is just to edit those handler functions, recompile with `cargo run`, and see your new window behavior immediately on screen.