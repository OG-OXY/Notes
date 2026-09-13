To get Steam and Gamescope working smoothly on NixOS with a hybrid setup ([Nvidia GTX 1070](https://www.google.com/search?ibp=oshop&prds=pvt:hg,pvo:48,mid:576462787486755241,imageDocid:156470587878170786,gpcid:17788173665730065684,headlineOfferDocid:14432916979591675483,catalogid:7861681919984318330,productDocid:14161089777717439574,rds:PC_17788173665730065684%7CPROD_PC_17788173665730065684&q=product&sa=X&ved=2ahUKEwiii5HYirGWAxXRM1kFHZAjDaYQxa4PegoIAAgACAIIChAC) + Ryzen iGPU), you need specific environment variables to force offloading and correct rendering backends. Put system-wide variables inside your `/etc/nixos/configuration.nix` and runtime variables directly into your Steam game launch options. 

(https://www.reddit.com/r/linux_gaming/comments/1ezz3tx/is_having_an_igpu_with_a_dgpu_just_a_complete/), [2](https://www.reddit.com/r/NixOS/comments/1ri2dvk/psa_you_can_set_global_environment_variables_for/), [3](https://wiki.nobaraproject.org/graphics/nvidia/gpu-selection-in-igpu-plus-dgpu-setup), [4](https://nixos.wiki/wiki/Steam)]

Where to Put the Configuration

Place your base configuration declarations inside your main **NixOS configuration file** (`/etc/nixos/configuration.nix`): 
(https://nixos.wiki/wiki/Steam), (https://www.reddit.com/r/NixOS/comments/1ri2dvk/psa_you_can_set_global_environment_variables_for/)]

```

{ config, pkgs, ... }: {
  # Enable hardware graphics and 32-bit support required by Steam
  hardware.graphics = {
    enable = true;
    enable32Bit = true;
  };

  # Enable Nvidia drivers
  services.xserver.videoDrivers = [ "nvidia" ];
  hardware.nvidia = {
    modesetting.enable = true;
    # Use open source or proprietary kernel modesetting depending on your preference; 
    # GTX 1070 works well on standard proprietary drivers
  };

  # Enable Steam and integrate Gamescope support
  programs.steam = {
    enable = true;
    gamescopeSession.enable = true;
  };

  # Enable Gamescope system-wide (ensure capSysNice is false to avoid crashes on Nvidia)
  programs.gamescope = {
    enable = true;
    capSysNice = false; 
  };
}

```


Essential Environment Variables

Because you are combining an older Nvidia architecture (GTX 1070 doesn't fully support modern Wayland/drm-leasing features the same way RTX/AMD cards do) with a Ryzen iGPU, Gamescope often gets confused about which processor should composite or render. (https://www.reddit.com/r/linux_gaming/comments/1ezz3tx/is_having_an_igpu_with_a_dgpu_just_a_complete/), (https://github.com/ValveSoftware/gamescope/issues/1643

Add these session variables in your `configuration.nix` under `environment.sessionVariables`, or handle them dynamically:

- `__NV_PRIME_RENDER_OFFLOAD=1` — Forces rendering tasks onto your Nvidia GTX 1070.

- `__GLX_VENDOR_LIBRARY_NAME=nvidia` — Routes GLX requests to the Nvidia driver stack.

- `GBM_BACKEND=nvidia-drm` — Tells the Generic Buffer Management layer to use Nvidia's DRM backend.

- `WLR_NO_HARDWARE_CURSORS=1` — Prevents invisible or flickering double cursors common in hybrid Wayland environments. (https://www.reddit.com/r/linux_gaming/comments/1ezz3tx/is_having_an_igpu_with_a_dgpu_just_a_complete/), [2](https://github.com/ValveSoftware/gamescope/issues/1643), [3](https://wiki.nobaraproject.org/graphics/nvidia/gpu-selection-in-igpu-plus-dgpu-setup), [4](https://discourse.nixos.org/t/gamescope-stopped-working/68293)]

Steam Launch Options for Games

Instead of wrapping the entire Steam client (which often breaks under hybrid Nvidia setups), call Gamescope on a **per-game basis** via Steam's properties menu. [[1](https://discourse.nixos.org/t/launching-steam-games-with-gamescope-and-mangohud/23936), [2](https://linuxgamecast.com/2023/03/gamescope-with-nvidia/)]

Right-click your game in Steam, go to **Properties > General > Launch Options**, and paste: [[1](https://github.com/NixOS/nixpkgs/issues/162562), [2](https://www.reddit.com/r/linux_gaming/comments/1cp3jcd/how_to_force_some_games_to_use_my_igpu_instead_of/)]

bash

```
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia gamescope -W 1920 -H 1080 -r 60 -e -- %command%
```


_(Adjust `-W`, `-H`, and `-r` to match your desired resolution and refresh rate)._ [[1](https://discourse.nixos.org/t/gamescope-refuses-to-work-with-steam/71417), [2](https://www.youtube.com/watch?v=wcs7JsMLHFY&t=282), [3](https://www.reddit.com/r/NixOS/comments/1lgxq3a/my_solution_to_wayland_steamgamescope_hell/)]

If you are running into a specific error message (such as a **crash on startup**, **error code 6**, or a **black screen**), tell me the **exact error** or your **display manager/window manager (KDE, GNOME, Hyprland, etc.)** so I can fine-tune the backend flags.

```
programs.steam = {
  enable = true;
  remotePlay.openFirewall = true; # Open ports in the firewall for Steam Remote Play
  dedicatedServer.openFirewall = true; # Open ports in the firewall for Source Dedicated Server
  localNetworkGameTransfers.openFirewall = true; # Open ports in the firewall for Steam Local Network Game Transfers
};
```

To use Gamescope with an older NVIDIA GTX 1070 and a Ryzen integrated GPU on NixOS, you need to define specific hybrid/offload environment variables in your global system configuration and invoke Gamescope with the right backend flags.Where to put themAdd the environment variables to your /etc/nixos/configuration.nix under environment.sessionVariables so they apply system-wide, and ensure Steam is enabled properly

Essential Gamescope Launch Flags for SteamWhen running gamescope inside Steam or launching a game via Steam launch options, pass the explicit credentials to force usage through the NVIDIA path:
bash
```
gamescope -W 1920 -H 1080 -r 60 -e -- %command%
```
(If embedding Gamescope inside an existing Wayland/X11 session fails, you may need to append --backend sdl or execute it directly from a TTY session).If you are experiencing a specific error (like a black screen, immediate segmentation fault, or WSI/Vulkan errors), please share the log output so I can help adjust the parameters or suggest a package overlay.

To run Steam with Gamescope on NixOS, use ENABLE_GAMESCOPE_WSI=1 for Vulkan application bridging, and specify DXVK_HUD=fps or AMD/Nvidia ICD loader variables if needed. Set these variables inside your NixOS configuration under programs.steam.gamescopeSession.env or in your user/system environment files.Essential Environment Variables

ENABLE_GAMESCOPE_WSI=1: Forces the Vulkan WSI layer to route through Gamescope for proper scaling and HDR.AMD_VULKAN_ICD=RADV: Forces the RADV driver if you use AMD hardware.DXVK_HUD=fps:
Optional performance overlay tracker for debugging.
Where to Put ThemOption 1: Dedicated Gamescope Session ModuleIf you use the built-in Gamescope session manager in NixOS, declare the variables right inside your configuration.nix

```
programs.steam = {
  enable = true;
  gamescopeSession = {
    enable = true;
    env = {
      ENABLE_GAMESCOPE_WSI = "1";
    };
    args = [
      "-W" "2560"
      "-H" "1440"
      "-r" "144"
    ];
  };
};
```

Option 2: Global System EnvironmentIf you run Steam nested or outside a dedicated Gamescope session, add the variables to your global environment block:nixenvironment.variables = {
  ENABLE_GAMESCOPE_WSI = "1";
};
Use code with caution.Option 3: Steam Game Launch OptionsIf a specific game requires passing flags or variables directly through individual properties in the Steam client, prepend the command line:bashENABLE_GAMESCOPE_WSI=1 gamescope -e -W 1920 -H 1080 -- %command%
Use code with caution.If you tell me whether you are trying to build a dedicated Gamescope session (SteamOS style) or run it nested inside a regular desktop environment, I can provide the exact configuration blocks you need.

``
To run Steam or gamescope smoothly on NixOS, configure your settings directly in your configuration.nix file using the native programs.steam and programs.gamescope options rather than manually exporting shell environment variables.Required Configuration OptionsAdd these


programs = {
  steam = {
  enable = true;
  gamescopeSession.enable = true; # Enables integrated gamescope session support
};
gamescope = {
  enable = true;
  enableWsi = true;
  capSysNice = false;
};

☆Key Environment Variables & Placement

If you need specific environment variables passed to Gamescope or a dedicated Gamescope session, place them inside your NixOS module definition rather than an external profile file:

* **Where:** Inside `programs.steam.gamescopeSession.env` or globally via `environment.variables`.
* **Common Variables to Include:**
  * `SDL_VIDEODRIVER = "wayland";` (Forces SDL games to use Wayland natively inside gamescope)
  * `CLUTTER_BACKEND = "wayland";`

Example of setting session environment variables cleanly in NixOS:
```nix
programs.steam.gamescopeSession.env = {
  SDL_VIDEODRIVER = "wayland";
  QT_QPA_PLATFORM = "wayland";
};
```

<FollowUp>
If you are trying to launch **individual games** using Gamescope via Steam launch options instead of running a dedicated Gamescope session, let me know. I can share how to adapt your `extraEnv` or FHS container PATH parameters for individual titles!
</FollowUp>