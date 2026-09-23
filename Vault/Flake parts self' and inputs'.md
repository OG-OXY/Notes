Here is your Obsidian markdown note, cleaned up with proper syntax, accurate code block formatting, and structured headings.

---

# Understanding `self'` and `inputs'` in `flake-parts`

In **flake-parts**, `self'` and `inputs'` (pronounced *"self prime"* and *"inputs prime"*) exist to eliminate the single most annoying part of traditional Flakes: manually typing `x86_64-linux` or passing system strings everywhere.

They are **system-bound snapshots** of your Flake’s `self` and `inputs`.

## The Fundamental Difference

| Object | System-Agnostic? | Example Usage |
| --- | --- | --- |
| `self` | **Yes** (Global) | `self.nixosModules.default` |
| **`self'`** | **No** (Bound to current system) | `self'.packages.myApp` |
| `inputs` | **Yes** (Global) | `inputs.nixpkgs.legacyPackages` |
| **`inputs'`** | **No** (Bound to current system) | `inputs'.nixpkgs.packages` |

---

## How They Work Under the Hood

When you define a `perSystem` block in `flake-parts`, it automatically inspects the current `system` (e.g., `x86_64-linux`) for that iteration and wraps `self` and `inputs` into prime equivalents:

* `inputs.nixpkgs.legacyPackages.x86_64-linux.hello` $\rightarrow$ **`inputs'.nixpkgs.packages.hello`** (or `pkgs.hello`)
* `self.packages.x86_64-linux.myCustomApp` $\rightarrow$ **`self'.packages.myCustomApp`**

The `x86_64-linux` attribute disappears completely because `flake-parts` handles system lookup behind the scenes.

---

## 1. Using Them Inside `perSystem`

Inside a `perSystem` function, `self'` and `inputs'` are available right in the argument list alongside `pkgs` and `system`:

```nix
{ inputs, ... }: {
  systems = [ "x86_64-linux" "aarch64-linux" ];

  perSystem = { pkgs, self', inputs', ... }: {
    # Define a custom package using an input's package directly
    packages.default = pkgs.stdenv.mkDerivation {
      name = "my-wrapper";
      # Pulling directly from an input without specifying architecture:
      buildInputs = [ inputs'.hyprland.packages.hyprland ];
    };

    # Reference another package defined inside this exact same Flake:
    devShells.default = pkgs.mkShell {
      inputsFrom = [ self'.packages.default ];
    };
  };
}

```

---

## 2. Using Them in NixOS Modules

When writing NixOS modules in a dendritic setup, you can inject `self'` and `inputs'` into your module scope using `specialArgs`.

### Step A: Pass System-Bound Inputs from `flake.nix`

Extract system-bound context using `inputs.self.perSystem.${system}` and pass it down:

```nix
# flake.nix
{ inputs, ... }: {
  systems = [ "x86_64-linux" ];
  imports = [ ./parts/nixos.nix ];

  perSystem = { system, ... }: {
    _module.args.pkgs = import inputs.nixpkgs {
      inherit system;
      config.allowUnfree = true;
    };
  };

  flake.nixosConfigurations.desktop =
    let
      system = "x86_64-linux";
    in
    inputs.nixpkgs.lib.nixosSystem {
      inherit system;
      # Pass self' and inputs' directly as specialArgs!
      specialArgs = {
        self' = inputs.self.perSystem.${system};
        inputs' = inputs.flake-parts.lib.inputsWithSystem system inputs;
      };
      modules = [ ./hosts/desktop ];
    };
}

```

### Step B: Consume Them Cleanly Inside Submodules

Any submodule in your tree can pull `self'` or `inputs'` directly from its argument header—without importing global `inputs` or caring about the system architecture:

```nix
# modules/desktop/hyprland.nix
{ config, pkgs, self', inputs', ... }:

{
  environment.systemPackages = [
    # Grab Hyprland directly from the flake input bound to target architecture
    inputs'.hyprland.packages.hyprland

    # Grab a custom package built elsewhere in your own flake
    self'.packages.my-custom-script
  ];
}

```

---

## 3. Passing `self'` and `inputs'` into Home Manager

To pass `self'` and `inputs'` into Home Manager's `extraSpecialArgs`, extract the system-bound context at the host declaration level.

### Option A: Home Manager as a NixOS Module (Most Common)

```nix
# parts/nixos.nix
{ inputs, ... }: {
  flake.nixosConfigurations.desktop =
    let
      system = "x86_64-linux";
      # Extract system-bound instances
      self' = inputs.self.perSystem.${system};
      inputs' = inputs.flake-parts.lib.inputsWithSystem system inputs;
    in
    inputs.nixpkgs.lib.nixosSystem {
      inherit system;
      specialArgs = { inherit self' inputs'; };
      modules = [
        inputs.home-manager.nixosModules.home-manager
        {
          home-manager = {
            useGlobalPkgs = true;
            useUserPackages = true;
            # Inject self' and inputs' into all Home Manager sub-modules
            extraSpecialArgs = { inherit self' inputs'; };

            users.ty = import ../home/ty.nix;
          };
        }
        ../hosts/desktop
      ];
    };
}

```

### Option B: Standalone Home Manager (`homeConfigurations`)

```nix
# parts/home.nix
{ inputs, ... }: {
  flake.homeConfigurations."ty@desktop" =
    let
      system = "x86_64-linux";
      self' = inputs.self.perSystem.${system};
      inputs' = inputs.flake-parts.lib.inputsWithSystem system inputs;
    in
    inputs.home-manager.lib.homeManagerConfiguration {
      pkgs = self'.pkgs; # Re-use the perSystem-configured pkgs instance
      
      # Inject self' and inputs' globally into HM modules
      extraSpecialArgs = { inherit self' inputs'; };

      modules = [
        ../home/ty.nix
      ];
    };
}

```

### Step C: Consume `self'` and `inputs'` in Home Manager Sub-Modules

Once injected into `extraSpecialArgs`, any nested file inside your Home Manager configuration can access `self'` and `inputs'` directly:

```nix
# home/modules/desktop-apps.nix
{ config, pkgs, self', inputs', ... }:

{
  home.packages = [
    # Pull a package directly without specifying 'x86_64-linux'
    inputs'.ghostty.packages.default

    # Pull a custom package defined in your own flake's perSystem outputs
    self'.packages.my-custom-script
  ];

  # Referencing custom packages from input flakes (e.g. NVF)
  programs.neovim.package = inputs'.nvf.packages.default;
}

```

---

## Core Benefits

* **Architecture Neutrality:** Your configuration files contain zero hardcoded `x86_64-linux` or `aarch64-linux` strings. Compiling for an ARM device resolves the ARM package variants automatically.
* **Evaluation Parity:** Home Manager shares the exact same pre-configured `pkgs` instance evaluated by your host system, preventing Nixpkgs from evaluating duplicate package sets and reducing memory footprint.
* **Scope Safety:** No complex attribute chaining or string interpolation inside deep nested submodules.
  
  [[Turning Main And Dendritic Host Into Modules While Maintaining Build Integrity And Function Of Main Config]]