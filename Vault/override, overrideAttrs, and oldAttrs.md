Here is the cleaned-up note formatted for Obsidian, utilizing callouts, proper tag organization, code block formatting, and internal wikilinks (`[[]]`) for seamless integration into your vault.

---

# Nix Overrides & `oldAttrs` Explained

#nix #nixpkgs #devops #configuration #learning

Understanding how `.override`, `.overrideAttrs`, and the `oldAttrs` pattern work in Nix derivation customizations.

---

## The `old` / `oldAttrs` Pattern

The variable name `old` or `oldAttrs` is **not magic or built-in Nix shorthand**—it is simply a standard function parameter name representing the package's existing derivation attributes.

Using `oldAttrs` allows you to safely inherit, modify, or extend pre-existing attributes (e.g., `cmakeFlags`, `postInstall`, `buildInputs`) rather than accidentally overwriting and wiping them out.

### How `oldAttrs` Protects Builds

Consider this standard pattern:

```nix
cmakeFlags = (oldAttrs.cmakeFlags or [ ]) ++ [
  "-DCMAKE_CUDA_ARCHITECTURES=61"
];

```

* **`oldAttrs`**: The variable bound to the un-overridden derivation attributes.
* **`oldAttrs.cmakeFlags`**: Fetches the original list of CMake flags assigned by upstream [[Nixpkgs]].
* **`or [ ]`**: A safety fallback returning an empty list if `cmakeFlags` wasn't previously defined.
* **`++ [ ... ]`**: Appends custom flags to the end of the existing list.

> [!important] Why This Matters
> If upstream [[Nixpkgs]] updates a package and adds necessary default flags, using `oldAttrs` **preserves** those updates automatically while appending your changes. Hardcoding `cmakeFlags` without `oldAttrs` completely wipes out upstream defaults, frequently breaking builds.

---

## Overriding Strategies: `.override` vs. `.overrideAttrs`

Depending on whether a package requires high-level feature flags or low-level build attribute tweaks, you will use one or both override methods.

### 1. High-Level Function Arguments: `.override`

* **Targets:** The Nix expression/function that defines the package.
* **Use Case:** Toggling feature flags exposed by the package maintainer (e.g., `cudaSupport = true`, `enableWayland = true`).

### 2. Derivation Attributes: `.overrideAttrs`

* **Targets:** The low-level `stdenv` build attributes after the package function evaluates.
* **Use Case:** Tweaking build steps, injecting patches, or altering build flags (e.g., `postInstall`, `cmakeFlags`, `patches`).

---

## Comparative Breakdown

| Package Type | Approach | Reason |
| --- | --- | --- |
| **Obsidian** | `.overrideAttrs` only | Prebuilt binary; doesn't take functional build arguments. Only requires a derivation-level tweak (e.g., wrapping `postInstall`). |
| **Ollama / Llama-cpp** | `.override` **+** `.overrideAttrs` | Source build; requires toggling package-level options (`cudaSupport`) *and* passing explicit low-level compilation options (`-DCMAKE_CUDA_ARCHITECTURES`). |

### Chained Example Syntax

```nix
(ollama.override {
  cudaSupport = true;
}).overrideAttrs (oldAttrs: {
  cmakeFlags = (oldAttrs.cmakeFlags or [ ]) ++ [
    "-DCMAKE_CUDA_ARCHITECTURES=61"
  ];
})

```

[[Turning Main And Dendritic Host Into Modules While Maintaining Build Integrity And Function Of Main Config]]