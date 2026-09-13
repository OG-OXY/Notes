In **flake-parts**, `self'` and `inputs'` (pronounced _"self prime"_ and _"inputs prime"_) exist to eliminate the single most annoying part of traditional Flakes: **manually typing `x86_64-linux` or passing `system` strings everywhere.**

They are system-bound snapshots of your Flake’s `self` and `inputs`.

### The Fundamental Difference

|Standard Flake Object|System-Agnostic?|Example Usage|
|---|---|---|
|`self`|Yes (Global)|`self.nixosModules.default`|
|`**self'**`|**No (Bound to current system)**|`self'.packages.myApp`|
|`inputs`|Yes (Global)|`inputs.nixpkgs.legacyPackages`|
|`**inputs'**`|**No (Bound to current system)**|`inputs'.nixpkgs.packages`|

### How They Work Under the Hood

When you define a `perSystem` block in `flake-parts`, `flake-parts` automatically inspects the current `system` (e.g., `x86_64-linux`) for that iteration and wraps `self` and `inputs` into prime equivalents:

- `**inputs.nixpkgs.legacyPackages.x86_64-linux.hello**` becomes `**inputs'.nixpkgs.packages.hello**` (or `pkgs.hello`)
- `**self.packages.x86_64-linux.myCustomApp**` becomes `**self'.packages.myCustomApp**`

Notice how the `x86_64-linux` attribute disappears completely. `flake-parts` handles the attribute lookup behind the scenes.

### 1. Using Them Inside `perSystem`

Inside a `perSystem` function, `self'` and `inputs'` are available right in the function argument list alongside `pkgs` and `system`:

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

### 2. Using Them in NixOS / Home Manager Modules

When writing NixOS or Home Manager modules in a dendritic setup, you can inject `self'` and `inputs'` into your module scope using `specialArgs` or `extraSpecialArgs`.

#### Step A: Pass System-Bound Inputs from `flake.nix`

Use `inputs.self.perSystem.${system}` to pass the system-bound context into your host configuration:

# flake.nix
{ inputs, ... }: {
  systems = [ "x86_64-linux" ];

  imports = [ ./parts/nixos.nix ];

  perSystem = { pkgs, ... }: {
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

#### Step B: Consume Them Cleanly Inside Your Submodules

Now, any submodule in your tree can pull `self'` or `inputs'` directly from its argument header—without importing `inputs` or caring about system architecture:

# modules/desktop/hyprland.nix
{ config, pkgs, self', inputs', ... }:

{
  environment.systemPackages = [
    # Grab Hyprland directly from the flake input bound to x86_64-linux
    inputs'.hyprland.packages.hyprland

    # Grab a custom package built elsewhere in your own flake
    self'.packages.my-custom-script
  ];
}

### Why This Design Prevents Scope Issues

1. **No Hardcoded System Chains:** You never have to write `.x86_64-linux` inside deep submodules. If you compile the host for ARM (`aarch64-linux`), `self'` automatically resolves ARM packages without altering a single module file.
2. **Single Source of Truth:** Your package outputs (`self'.packages`) and input dependencies (`inputs'.<name>.packages`) are evaluated cleanly once per system.

To pass `self'` and `inputs'` into Home Manager's `extraSpecialArgs` using `flake-parts`, you extract the system-bound context from `inputs.self.perSystem.${system}` and `inputs.flake-parts.lib.inputsWithSystem system inputs` at the host declaration level.

This allows every sub-module in your Home Manager configuration tree to consume `self'` and `inputs'` directly in its header arguments without specifying architecture strings or re-evaluating inputs.

### Step 1: Wire `extraSpecialArgs` in Your Host Configuration

Whether you run Home Manager as a **standalone flake output** or as a **NixOS module** via `home-manager.nixosModules.home-manager`, configure the arguments where the Home Manager instance is instantiated:

#### Option A: Home Manager as a NixOS Module (Most Common)

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

#### Option B: Standalone Home Manager (`homeConfigurations`)

If you build standalone Home Manager profiles via `home-manager.lib.homeManagerConfiguration`:

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

### Step 2: Consume `self'` and `inputs'` in Home Manager Sub-Modules

Once injected into `extraSpecialArgs`, any nested module or file inside your Home Manager setup can access `self'` and `inputs'` directly from its top-level arguments:

# home/modules/desktop-apps.nix
{ config, pkgs, self', inputs', ... }:

{
  home.packages = [
    # Pull a package directly from a flake input without specifying 'x86_64-linux'
    inputs'.ghostty.packages.default

    # Pull a custom package/script defined in your own flake's perSystem outputs
    self'.packages.my-custom-script
  ];

  # Example: Referencing custom Neovim or Hyprland packages from inputs
  programs.neovim.package = inputs'.nvf.packages.default;
}

### Why This Workflow Works Cleanly

1. **Architecture Neutrality:** Your Home Manager files do not contain hardcoded `x86_64-linux` strings. If you compile your configuration for an ARM device (`aarch64-linux`), `self'` and `inputs'` resolve the ARM package variants seamlessly.
2. **Evaluation Parity:** Home Manager shares the exact same pre-configured `pkgs` and system dependencies evaluated by your host system, preventing Nixpkgs from evaluating duplicate package sets.