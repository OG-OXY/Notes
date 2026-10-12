You have reached a classic fork in the road of high-level Nix architecture. Given your strict no-abstraction, anti-framework philosophy, you are going to want to move away from devenv.nix / YAML and transition to native devShells using raw Nix flakes.
Here is the breakdown of why you should do this, how it works structurally, and exactly where to declare them to fix your multi-project dependency headaches.
------------------------------
## What is Superior and Why? (devenv vs. Native devShells)
Native devShells are superior for your philosophy.
devenv is a wrapper framework. Under the hood, all it does is generate native Nix code for you. It relies heavily on module systems, abstractions, and standard patterns to make things "easy" for beginners.
By writing native devShells, you strip away that layer of magic. You get complete control over environmental variables, package overlays, compiler flags, and shell hooks, without a wrapper tool trying to manage your YAML configurations.
------------------------------
## Where Do You Declare Them? (Scope & Architecture)
You mentioned choosing between your root flake, your nixosConfigurations, or separate directories. Understanding the difference is crucial for your system's architecture.
## 1. In Your Local Project Directories (The standard way)
You create a flake.nix directly inside your ~/projects/my-rust-app/ folder. When you open a terminal in that folder and type nix develop, Nix looks for the devShells output in that specific directory's flake.

* How it works: It is completely isolated to that project.
* The benefit: This is how you solve your problem with massive runtime libraries and conflicting dependencies. Project A and Project B can require totally different, conflicting C libraries or Rust toolchains because their local flakes explicitly control their individual shells.

## 2. In Your Global System Flake (Under the Root outputs)
If you want global development environments that you can spin up from anywhere on your PC, you declare them as outputs in your main system configuration flake, but not inside nixosConfigurations.
Your system flake has a top-level outputs attribute. nixosConfigurations is just one of those outputs (meant for building your OS). devShells is a sibling output.

# High-level architecture of your root system flake
{
  inputs = { ... };

  outputs = inputs@{ self, nixpkgs, flake-parts, ... }: 
  flake-parts.lib.mkFlake { inherit inputs; } {
    
    # This output builds your actual OS
    flake.nixosConfigurations.myHost = ...; 

    # This output defines your globally accessible shells
    perSystem = { pkgs, ... }: {
      devShells = {
        rust-global = pkgs.mkShell {
          buildInputs = [ pkgs.rustc pkgs.cargo ];
        };
        python-global = pkgs.mkShell {
          buildInputs = [ pkgs.python3 ];
        };
      };
    };
  };
}

## Can I enter these from anywhere on my PC?
Yes, but you have to tell Nix where to find them.
If you are sitting in your ~/Downloads directory, typing just nix develop will fail because there is no flake there. However, you can reference your global system flake from anywhere by passing its path or a URI.
If your system configuration flake lives in ~/dotfiles, you can enter your global rust shell from any folder by typing:

nix develop ~/dotfiles#rust-global

(If using Nushell or Fish, you can easily alias this to developer-rust or similar).
------------------------------
## Handling Huge Runtime Libs & Project Dependencies
You mentioned struggling with projects requiring massive runtime libraries and different Rust versions. The clean, native way to handle this without framework clutter is by utilizing Project-Isolated Flakes combined with direnv.
Instead of trying to maintain one massive, multi-language global shell that breaks when dependencies conflict, you should give each complex project its own localized flake.nix.
To inject heavy system or C libraries (like openssl, glib, or X11 libs) so that Cargo/Rust can actually find them during compilation, use shellHook and LD_LIBRARY_PATH. Here is an explicit, raw example:

# ~/projects/complex-rust-app/flake.nix
{
  inputs.nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

  outputs = { self, nixpkgs }: let
    system = "x86_64-linux";
    pkgs = nixpkgs.legacyPackages.${system};
    
    # Explicitly list the heavy runtime libraries your Rust code compiles against
    runtimeLibs = with pkgs; [
      openssl
      glib
      xorg.libX11
    ];
  in {
    devShells.${system}.default = pkgs.mkShell {
      # Tools you need executable in the shell path
      buildInputs = with pkgs; [
        rustc
        cargo
        pkg-config # Crucial for Rust to locate C libraries
      ] ++ runtimeLibs;

      # Env variables mapped explicitly so Rust/Cargo can find the runtimes
      shellHook = ''
        export LD_LIBRARY_PATH="${pkgs.lib.makeLibraryPath runtimeLibs}:$LD_LIBRARY_PATH"
      '';
    };
  };
}

## The Ultimate Speedrun Workflow (Pure Nix + Direnv)
To make this completely seamless so you don't even have to type nix develop:

   1. Keep using direnv (it is an independent binary tool, not a bloated Nix framework).
   2. Inside your isolated project folder, create a .envrc file with a single line: use flake
   3. The moment you cd into that project directory via Fish or Nushell, direnv instantly triggers the native devShell from that directory's flake.nix. All dependencies, heavy libraries, and exact toolchains seamlessly load into your shell environment, and instantly vanish when you cd out.

Are you currently using the standard Rust compiler from nixpkgs, or do you need to pin specific nightly/stable toolchain versions using toolchain files (like rust-toolchain.toml) for your projects? If you need strict version pinning across your projects, I can show you how to tie the nix-community rust-overlay directly into your native devShell setup.

You make an excellent point. It is crucial to recognize when a framework is actively doing heavy lifting versus when it is just adding useless bloat.

Your `devenv.nix` configuration is incredibly clean and demonstrates exactly why `devenv` can be powerful: it abstracts away the massive amount of flake inputs and manual plumbing required to set up components like `rust-analyzer` with the correct standard library path (`rust-src`).

To achieve the exact same environment using a pure native flake without `devenv`, you would have to pull in the `rust-overlay` input, handle the system architecture mapping manually, and explicitly wire up the toolchain components. For comparison, here is what your exact setup looks like translated into a native, standalone `flake.nix`:

```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    rust-overlay.url = "github:oxalica/rust-overlay";
  };

  outputs = { self, nixpkgs, rust-overlay }:
    let
      system = "x86_64-linux"; # Or mapped via flake-parts / lib
      pkgs = import nixpkgs {
        inherit system;
        overlays = [ (import rust-overlay) ];
      };

      gbmPkg = pkgs.mesa-libgbm or pkgs.libgbm or pkgs.mesa;
      runtimeLibs = with pkgs; [
        gbmPkg wayland wayland-protocols libxkbcommon libinput libdisplay-info
        udev dbus pixman mesa libdrm seatd fontconfig vulkan-loader libglvnd
        egl-wayland xorg.libX11
      ];

      libPath = pkgs.lib.makeLibraryPath runtimeLibs;
      pkgConfigPath = pkgs.lib.makeSearchPathOutput "dev" "lib/pkgconfig" runtimeLibs 
        + ":" + pkgs.lib.makeSearchPathOutput "out" "lib/pkgconfig" runtimeLibs 
        + ":" + pkgs.lib.makeSearchPath "share/pkgconfig" runtimeLibs;

      # The native equivalent to languages.rust.components
      rustToolchain = pkgs.rust-bin.stable.latest.default.override {
        extensions = [ "rust-src" "rust-analyzer" "clippy" "rustfmt" ];
      };
    in
    {
      devShells.${system}.default = pkgs.mkShell {
        buildInputs = [
          rustToolchain
          pkgs.pkg-config pkgs.bacon pkgs.clang pkgs.lld pkgs.gdb pkgs.foot
        ] ++ runtimeLibs;

        shellHook = ''
          export LIBRARY_PATH="${libPath}"
          export LD_LIBRARY_PATH="${libPath}"
          export PKG_CONFIG_PATH="${pkgConfigPath}"
          clear
          export SHOW_PROJECT_GREETING=1
        '';
      };
    };
}
```

## The Verdict: Stick with `devenv` Here

Seeing them side-by-side confirms your instinct: `devenv` is actually doing a useful job here.

- It completely eliminates the boilerplate of importing external overlays (`rust-overlay`) just to fetch components like `rust-src` or `rust-analyzer`.
- It cuts down on structural noise, keeping your project files concise.

Since you are already comfortable with high-level Nix and understand exactly what `devenv` is doing under the hood, using it as a localized tool to spin up project environments is completely fine. It is an isolated utility, not a system-wide framework that dictates your entire OS architecture.

---

## How to Fix Your Multi-Project Dependency Headaches

Your main issue right now isn't the framework itself—it's scope and isolation. You mentioned struggling because some projects require massive, different runtime libraries or entirely separate Rust compiler versions, making it hard to know how to handle them.

The solution is simple: One `devenv.nix` file per project directory.

Never try to create a "global" `devenv` folder or use a single development environment for multiple different projects.

1. Complete Project Isolation: When you start a new Rust project (e.g., `~/projects/graphics-app`), run `devenv init` directly inside that specific folder.
2. Tailor-Made Environments: If _Project A_ needs the massive Wayland/Mesa runtime libraries you pasted above, include them in `~/projects/project-a/devenv.nix`. If _Project B_ is a simple command-line utility that only needs standard libraries, keep `~/projects/project-b/devenv.nix` dead simple without the graphics stack.
3. Rust Version Pinning: If a specific project requires Rust Nightly features, you change that single project's file to `channel = "nightly";`.

Because `devenv` stores all downloaded packages inside isolated store paths linked to that specific directory, the environments will never bleed into each other or conflict.

## Seamless Shell Integration with Nushell & Fish

To completely avoid having to manage these environments manually or think about them when you switch tasks, ensure you are utilizing `direnv`.

1. Install `direnv` and hook it into your `fish` or `nushell` configuration.
2. In the root directory of your Rust project, alongside your `devenv.nix`, create a file named `.envrc` containing exactly one line:
    
    ```bash
    use devenv
    ```
    
3. Run `direnv allow` once.

The moment you use `cd` in Fish or Nushell to enter your project folder, `direnv` instantly looks at your `devenv.nix`, builds the environment silently, sets your `LD_LIBRARY_PATH` and `PKG_CONFIG_PATH`, and exposes your exact isolated Rust compiler. The moment you `cd` out, your shell instantly clean-slates back to normal.

Since you are using Nushell alongside Fish, are you currently using the native Nushell hooks for `direnv` (like `nix-direnv` or `nu-env`), or are you running into issues where environment variables like `LD_LIBRARY_PATH` don't pass correctly through Nushell's structured pipelines? I can help you verify that your shell is catching your project variables perfectly.