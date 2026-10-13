Correct—in Nushell, standard **aliases are just direct invisible replacements**. When you type an `alias`, Nushell processes it under the hood like traditional Bash or Zsh aliases. You don't get that live visual expansion in the terminal where it replaces `gco` with `git checkout` as soon as you hit `Space`.

### Why Nushell doesn't do this out of the box

Fish handles `abbr` at the interactive shell level by intercepting your buffer text before it hits the parser.

Nushell handles command parsing differently: it parses your input into strongly typed **AST data structures** (tables, pipelines, records) before running it. Because it treats commands as structured code rather than raw strings, it doesn't have a built-in `abbr` command in its standard library.

---

### Can you make it expand like Fish?

**Yes, but you have to hook into Reedline** (Nushell's line editor).

If you want the exact Fish experience—where pressing `Space` auto-expands the word on your screen so you can edit it or see what it's doing—you have to write a small keybinding script in your `config.nu`.

Here is the community pattern for setting up Fish-style `abbr` in Nushell:

```shell
# 1. Define your abbreviations table
let abbrs = {
    "gco": "git checkout",
    "g": "git",
    "nr": "nixos-rebuild switch --flake .",
    "h": "herdr",
}

# 2. Add the spacebar keybinding to Reedline in config.nu
$env.config = {
    keybindings: [
        {
            name: expand_abbr
            modifier: none
            keycode: space
            mode: [emacs, vi_normal, vi_insert]
            event: {
                send: executehostcommand
                cmd: "
                    let line = (commandline)
                    let words = ($line | split row ' ')
                    let last_word = ($words | last)
                    
                    # Check if the last word is in our abbr table
                    if ($last_word in ($abbrs | columns)) {
                        let expanded = ($abbrs | get $last_word)
                        # Replace the last word with the expansion + space
                        commandline edit --replace ($line | str replace --regex $'($last_word)$' $'($expanded) ')
                    } else {
                        # Normal spacebar behavior
                        commandline insert ' '
                    }
                "
            }
        }
    ]
}

```

### The Takeaway

* **If you use `alias`:** It executes instantly, but **won't expand** visually in your terminal buffer.
* **If you want visual expansion (Fish style):** You have to use the Reedline `space` keybinding hook above.

If you love the visual buffer expansion where you can double-check complex flags before hitting `Enter`, Fish still has the cleanest native implementation. But if you want Nushell's structured pipelines and typed data, that Reedline snippet gives you the exact same `abbr` behavior!