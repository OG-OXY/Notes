---
id: nixosmodulemastery
aliases: []
tags: []
---
**nix-module-authoring-mastery.norg**









# NixOS Module Authoring Framework

When writing custom NixOS modules, every design decision boils down to how user inputs in `configuration.nix` are validated, transformed, and written to disk or system environment paths.
This guide breaks down the core architecture, contrast patterns, and internal execution order so you can build any module directly from upstream application documentation.



1. Core Architecture & Module Execution Lifecycle



Every NixOS module is a Nix function returning an attribute set divided into three fundamental parts:



```nix
{ config, lib, pkgs, ... }:

let
# 1. Private helpers, derivations, string transformations
in
{
# 2. Public interface (API contract exposed to configuration.nix)
options = { ... };

```
 # 3. System execution (What NixOS actually builds when enabled)
 config = lib.mkIf config.programs.mytool.enable { ... };

```

}

```




### 1.1 The Module Pipeline

1. **Evaluation Phase**: Nix reads `options` to construct the system option tree and assign default values.
2. **Merge Phase**: User settings from `configuration.nix` override defaults.
3. **Transformation Phase**: The `let` block transforms merged inputs into config files (`pkgs.writeText`) or scripts (`pkgs.writeShellScriptBin`).
4. **Realization Phase**: `config` injects generated packages or environment files into `/run/current-system/`.



--



2. Architectural Comparison: The Two Module Paradigms



Understanding when to choose **Explicit Options** versus **Freeform Attribute Sets** is the key to writing clean modules without unnecessary boilerplate.




### 2.1 Pattern A: Explicit Schemas (`wlr-which-key` Pattern)




```
Used when you must validate strict data types, provide sensible defaults, or perform **schema translation** (e.g., mapping a single `margin` setting into top/bottom/left/right fields).



- **Type Structure**: `lib.types.nullOr lib.types.int`, `lib.types.submodule`
- **Default Behavior**: Unset options evaluate to `null` or pre-defined default values.
- **Sanitization Required**: Must use `lib.filterAttrs (_: v: v != null)` before serializing to JSON/YAML so `null` values are stripped rather than written as `"field": null`.



```nix
# Sanitization Snippet
configSet = lib.filterAttrs (_: v: v != null) {
  inherit (cfg.settings) anchor font;
  margin_top = cfg.settings.margin;
  margin_bottom = cfg.settings.margin;
  margin_left = cfg.settings.margin;
  margin_right = cfg.settings.margin;
};

```



```




### 2.2 Pattern B: Freeform Attribute Sets (`warpd` Pattern)




```
Used when upstream software accepts standard key-value configuration formats (INI, plain text, simple JSON/TOML) and you want a lightweight module that automatically supports future upstream options without requiring module updates.



- **Type Structure**: `lib.types.attrsOf (lib.types.oneOf [ lib.types.str lib.types.int lib.types.bool ])`
- **Default Behavior**: Defaults to an empty set `{}`. No default values exist for individual keys.
- **Sanitization Required**: None. Unwritten keys simply do not exist in memory.



```nix
# Generator Snippet for key: value files (INI / warpd syntax)
formatLine = name: value:
  if builtins.isBool value then
    "${name}:${if value then "true" else "false"}"
  else
    "${name}:${toString value}";

configFile = pkgs.writeText "warpd-config" ''
  ${lib.concatStringsSep "\n" (lib.mapAttrsToList formatLine cfg.settings)}
'';

```



```




### 2.3 Comprehensive Comparison Matrix




 Feature | Explicit Schema (Pattern A) | Freeform Attribute Set (Pattern B) |
 --- | --- | --- |
 **Primary Use Case** | Complex apps, schema translation, strict structural requirements | Simple key-value configs, INI files, fast prototyping |
 **Editor Autocomplete** | Full LSP completion via `nixd`_`statix` | Limited to general type checking (string, int, bool) |
_ **NixOS Manual Docs** | Automatically generates option descriptions and defaults | Generates a single generic description for `settings` |
 **Upstream Compatibility** | Breaks if upstream changes key names unless module is updated | Supports new upstream keys instantly without code changes |
 **Null Handling** | Requires `nullOr` + `libfilterAttrs` | No `null` handling needed; unwritten keys don't exist |

--



3. Technical Reference: Nix Types & Functional Primitive Operators



To write modules without assistance, memorize these high-frequency Nix primitives.




### 3.1 Essential Types (`lib.types`)

- `lib.types.str`: Simple string.
- `lib.types.int`: Integer value.
- `lib.types.bool`: Boolean (`true` or `false`).
- `lib.types.nullOr T`: Accepts type `T` or `null`.
- `lib.types.listOf T`: Homogeneous list of type `T`.
- `lib.types.attrsOf T`: Attribute set where all values conform to type `T`.
- `lib.types.oneOf [ T1 T2 `: Accepts any of the listed primitive types.
- `lib.types.submodule { options = { ... }; `: Defines custom nested data structures.




### 3.2 High-Frequency Functional Operators

- `lib.mapAttrsToList (name: value: ...)`: Converts an attribute set into a list by evaluating a function over each key-value pair.
- `lib.filterAttrs (name: value: predicate)`: Returns a new attribute set containing only pairs where `predicate` evaluates to `true`.
- `lib.concatStringsSep separator list`: Joins a list of strings using a string delimiter (e.g., `"n"`).
- `inherit (set) key1 key2;`: Extracts `key1` and `key2` from `set` and assigns them to matching local variable names in the current scope.
- `builtins.toJSON set`: Serializes a Nix attribute set directly into valid JSON (which is also strictly valid YAML 1.2).




### 3.3 Target File Generation Utilities

- `pkgs.writeText "filename" "content"`: Writes plain text to a read-only store file.
- `pkgs.writeShellScriptBin "bin-name" ''content''`: Creates an executable bash script placed inside `/bin/bin-name` within the Nix store.



--



4. Complete Freeform Blueprint: `programs.warpd`



Save this full module implementation as your standard reference template for simple system modules.



```nix
{ config, lib, pkgs, ... }:

let
cfg = config.programs.warpd;

```
 # Transform Nix attribute sets into warpd's native key: value format
 formatLine = name: value:
   if builtins.isBool value then
     "${name}:${if value then "true" else "false"}"
   else
     "${name}:${toString value}";

 configFile = pkgs.writeText "warpd-config" ''
   # Generated by NixOS Module
   ${lib.concatStringsSep "\n" (lib.mapAttrsToList formatLine cfg.settings)}
 '';

```

in
{
options.programs.warpd = {
enable = lib.mkEnableOption "warpd modal mouse navigation daemon";

```
   package = lib.mkPackageOption pkgs "warpd" { };

   settings = lib.mkOption {
     type = lib.types.attrsOf (lib.types.oneOf [
       lib.types.str
       lib.types.int
       lib.types.bool
     ]);
     default = { };
     description = "Freeform key-value settings corresponding directly to warpd config syntax.";
     example = lib.literalExpression ''
       {
         hint_activation_key = "A-M-h";
         grid_activation_key = "A-M-g";
         speed = 400;
       }
     '';
   };
 };

 config = lib.mkIf cfg.enable {
   # Install binary to system PATH
   environment.systemPackages = [ cfg.package ];

   # Grant input event device permissions required for Wayland compositors
   users.groups.input.members = lib.optional (config.users.users ? ty) "ty";

   # Deploy configuration to system path
   environment.etc."warpd/config".source = configFile;
 };

```

}

```



--



5. Authoring Workflow Decision Tree



When building a module for a new CLI app or daemon, follow this decision path:



1. **Inspect Upstream Config Format**:

# Is it JSON/YAML? rightarrow Use `builtins.toJSON` + `pkgs.writeText`.


# Is it key: value / INI? rightarrow Use `lib.mapAttrsToList` + `lib.concatStringsSep "n"`.


# Is it CLI flags? rightarrow Build a script wrapper using `pkgs.writeShellScriptBin`.





2. **Determine Option Strictness**:

# Needs field mapping or complex lists? rightarrow Pattern A (Explicit options + `submodule`).


# Simple 1:1 key-value options? rightarrow Pattern B (Freeform `attrsOf`).





3. **Wire up System Execution**:

# Add packages to `environment.systemPackages`.


# Expose global configs to `environment.etc."app/config".source`.


# Configure system permissions (e.g., `users.groups.input`).

