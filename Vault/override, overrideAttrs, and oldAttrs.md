That `old` (or `oldAttrs`) variable isn't magic or shorthand—it’s just a **parameter name for a function argument** that holds the package's _existing_ attributes.

Using `old` or `oldAttrs` lets you safely inherit or append to whatever was already there (like existing `cmakeFlags` or `postInstall` scripts) instead of completely wiping them out.
### Why the Obsidian example looked simpler than your Ollama/llama-cpp examples:

They are doing two completely different levels of overriding based on what the package needs:
#### 1. Obsidian: Only needs `.overrideAttrs`

Obsidian is a prebuilt binary package. It doesn't take functional build arguments (like `cudaSupport = true`).
It only needs a derivation-level tweak to its install phase (`postInstall`) to add the wrapper flags.
- It doesn't have an argument function to `.override`, so you skip straight to `.overrideAttrs`.
#### 2. Ollama / Llama-cpp: Uses BOTH `.override` and `.overrideAttrs`

Your Ollama and llama-cpp examples use a chained combination because they require two different steps:
1. **`.override { ... }`**: This targets the **Nix package function** itself. It passes arguments _into_ the builder function before it evaluates (e.g., turning `cudaSupport = true` on, or configuring package-level options).
2. **`.overrideAttrs (...)`**: Once that function evaluates into a derivation, this targets the **stdenv builder attributes** (like `cmakeFlags`), allowing you to inject low-level compilation flags like `-DCMAKE_CUDA_ARCHITECTURES=61`
### What `oldAttrs` is actually doing for you:

Look closely at this line from your Ollama snippet:
Nix
```
cmakeFlags = (oldAttrs.cmakeFlags or [ ]) ++ [ "-DCMAKE_CUDA_ARCHITECTURES=61" ];
```
- **`oldAttrs`** is just the variable name holding the previous attributes.
- **`oldAttrs.cmakeFlags`** grabs whatever CMake flags Ollama was _already_ compiling with by default.
- **`or [ ]`** acts as a safety fallback in case the package didn't have any `cmakeFlags` defined natively.
- **`++ [...]`** safely tacks your custom CUDA architecture flag onto the end of the existing list.

Without `oldAttrs`, if upstream Nixpkgs updates Ollama tomorrow and adds a new default cmake flag, your override would completely overwrite and destroy it, causing a build failure. Using `oldAttrs` preserves upstream's defaults while letting you inject your specific changes.