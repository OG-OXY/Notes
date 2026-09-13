No, including `smithay` as a dependency in your `Cargo.toml` only gives you access to Smithay’s **library crates** (the API building blocks like `smithay::backend`, `smithay::wayland`, etc.). It **does not** compile or expose `anvil` as an executable or runnable module inside your project.

`anvil` lives inside Smithay's git repository under its `anvil/` subdirectory as an example crate.

Since you are using `devenv.nix`, here are two ways to get `anvil` into your workflow instantly without breaking your setup:

### Option 1: Git Submodule or Subdirectory (Recommended for Hacking)

If you want to read, modify, and break `anvil`'s code directly, clone the Smithay source right into your project folder.

1. In your project root, clone Smithay into a subfolder:
    
    git clone https://github.com/Smithay/smithay.git
    
2. Update your root `Cargo.toml` to turn your project into a Cargo Workspace that includes `anvil`:
    
    [workspace]
    members = [
        "smithay/anvil",
        "." # your current project
    ]
    
3. Run `anvil` directly using cargo through your `devenv` shell:
    
    cargo run --bin anvil -- --winit
    

Now you can open `smithay/anvil/src/` in Neovim, edit its source code, and run it inside your local dev environment.

### Option 2: Clone `anvil` Standalone Outside Your Project

If you prefer keeping your main project directory clean, clone Smithay somewhere else in your environment and launch `devenv shell` inside it.

git clone https://github.com/Smithay/smithay.git
cd smithay
devenv shell # or run directly if your system toolchain provides gcc/pkg-config/wayland libs
cargo run --bin anvil -- --winit

### Why Option 1 works so well with `devenv.nix`

`devenv.nix` provides all the necessary system dependencies (like `wayland`, `libxkbcommon`, `mesa`, `egl`, `fontconfig`, and `pkg-config`) that Smithay requires to compile C bindings and talk to your GPU.

When you run `cargo run --bin anvil -- --winit` inside your `devenv shell`, cargo will use your `devenv` toolchain to compile Anvil and launch it in a window right on top of your current Hyprland session.

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