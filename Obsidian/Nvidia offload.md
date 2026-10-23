GPU offloading allows your system to run display output and low-power desktop applications through the CPU's integrated graphics (**AMD Radeon Graphics** on your 7600X) while delegating demanding workloads—like Steam games—to your discrete **NVIDIA GTX 1070**.

### Step 1: Configure NixOS Hardware Profile

To enable PRIME offloading, configure the `hardware.nvidia` block in your system configuration (e.g., `configuration.nix` or hardware flake module):

hardware.nvidia = {
  modesetting.enable = true;
  powerManagement.enable = false;
  open = false; # Required for Pascal (GTX 1000 series)

  prime = {
    offload = {
      enable = true;
      enableOffloadCmd = true; # Adds 'nvidia-offload' helper script to PATH
    };

    # Bus IDs for your hardware
    amdgpuBusId = "PCI:00:01:0"; # Integrated 7600X iGPU
    nvidiaBusId = "PCI:01:00:0"; # Discrete GTX 1070 dGPU
  };
};

> **Note:** Identify exact PCI Bus IDs via terminal with `lspci | grep -E 'VGA|3D'`. Convert output like `01:00.0` into the Nix standard format `PCI:01:00:0`.

### Step 2: Offloading Applications & Games

Once the system is rebuilt (`nixos-rebuild switch`), offload graphics processing using environment variables or the built-in wrapper.

#### Option A: Using standard Steam Launch Options

In any Steam game's **Properties → Launch Options**, prepend the environment variables:

__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia %command%

Combine this with GameMode if installed:

__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia gamemoderun %command%

#### Option B: Terminal & External Apps

Launch any standalone binary or program on the GTX 1070 using the generated helper wrapper:

nvidia-offload <application_name>

### Step 3: Verification

Confirm render offloading is active on the GTX 1070 by running:

nvidia-offload glxinfo | grep "OpenGL vendor"

If offloading works correctly, the terminal output will explicitly state `**NVIDIA Corporation**` rather than `AMD`.

For a setup running NixOS with an AMD Ryzen 5 7600X and an NVIDIA GTX 1070 (Pascal architecture), optimal launch options balance system scheduling and GPU resource allocation.
Universal Launch Command Structure
In Steam game properties, string commands sequentially with %command% placed at the very end:
gamemoderun %command%

Hardware- & System-Specific Options
1. Gamemode (System Performance Optimization)
Forces your system into a high-performance profile (CPU governor, I/O priorities, and process niceness).
 * Launch Option: gamemoderun %command%
 * NixOS Requirement: Requires programs.gamemode.enable = true; in system configuration.
2. Hybrid / PRIME Offloading (If Using 7600X iGPU)
Because the 7600X includes an integrated GPU, Steam or Vulkan wrappers might attempt to initialize on the iGPU instead of the dGPU (GTX 1070). If a game runs on the wrong GPU:
 * Launch Option: __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia %command%
3. DXVK Async Pipeline State (Stutter Reduction)
Pascal GPUs like the GTX 1070 lack modern hardware ray tracing and native Mesh Shading, making Vulkan pipeline creation stutters more noticeable in DX11/DX12 games run via Proton.
 * Launch Option: RADV_PERFTEST=gpa %command% or DXVK_ASYNC=1 %command% (Requires custom GE-Proton builds supporting DXVK-Async)
4. Performance Overlay (MangoHud)
If tracking frametimes, CPU thread load, or GPU thermals:
 * Launch Option: mangohud %command%
 * NixOS Requirement: Add pkgs.mangohud to environment.systemPackages or Home Manager.
Summary of Best Launch Strings
| Workflow | Launch Command |
|---|---|
| Standard Baseline | gamemoderun %command% |
| Full Monitoring Stack | gamemoderun mangohud %command% |
| Force Discrete NVIDIA GPU | __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia gamemoderun %command% |
| DirectX 11 Fallback (For heavy DX12 Unreal Engine titles struggling on GTX 1070) | gamemoderun %command% -dx11 |
NixOS Configuration Prerequisite Check
Ensure the NixOS system modules for Steam and NVIDIA are declared in /etc/nixos/configuration.nix or your flake modules:
# NVIDIA Driver & Modesetting
services.xserver.videoDrivers = [ "nvidia" ];
hardware.nvidia = {
  modesetting.enable = true;
  powerManagement.enable = false;
  open = false; # GTX 1070 (Pascal) requires the proprietary closed driver module
};

# Steam & Gamemode Integration
programs.steam = {
  enable = true;
  remotePlay.openFirewall = true;
};
programs.gamemode.enable = true;

