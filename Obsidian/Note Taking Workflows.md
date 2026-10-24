
# Note Workflows
1. Which Ones The Best?
2. Weight Pros & Cons
3. Make A Decision

## Emacs & Org +Markdown & Pandoc ==> Org (Write/View/Explore)
- Hard
- Dont Know Syntax
- Hard To Navigate
- UGLY! (Viewing Experience)
- VERY GOOD ORGANIZATIONAL SYSTEM! (That I Dont Even Know How To Use)
- Convuluted
- Dont Know Any Plugins/How To Configure/Whats Available To Improve Viewing Experience
---

## Helix & Markdown (Write) +Pandoc ==> Org & Emacs (View/Explore)
- Second Worse?
- Makes Least Since.
- Mine Aswell Do Next Catagory With Obsidian View With Option To Export To Org And Use Org-Mode Agenda, Todo Lists, Diary Entries, And Ecosystem Made Available ETC.
---

## Helix & Markdown (Write) +Obsidian (View/Explore) +Markdown & Pandoc ==> Org & Emacs (View/Explore)
- Best Of All Worlds
- Seems Like Least Cons
- Could Ease Me Into Learning Org
- Would Need To Fix Obsidian Modal Breakage Though.

### Obsidian Modify Viewing Experience & Navigation To Solve Modal Problem
- Can You Use Native Obsidian Plugins?
- How To Add Plugins Via Nix If Not?
- Best Option? Best Viewing Experience, But Breaks Modal Workflow. 
---

## Helix & Markdown +Zk (Write/View/Explore)
- Possible 2ND 1ST Place?
- Completely Helix Native
- Mediocre Viewing Experience
- Way To Use This And Link To Open Obsidian To View After FZFing Database?

### Set Up
To replicate that exact terminal-native `zk` + Helix markdown notebook workflow from the video on your NixOS system, you only need a very lean stack.

Here are the packages required:

### 1. The Core CLI & Search Engines

* **`zk`** — The plaintext note-taking assistant and LSP server (`pkgs.zk`). This is the engine that handles indexing, templates, frontmatter generation, and the Language Server Protocol.
* **`ripgrep`** (`pkgs.ripgrep` or `pkgs.rg`) — Required under the hood by `zk` for lightning-fast text searching across your notes.
* **`fzf`** (`pkgs.fzf`) — Used by `zk` for interactive terminal pickers (`zk list -i`, `zk edit -i`).

### 2. The Text Editor & LSP Client

* **`helix`** (`pkgs.helix` or `pkgs.hx`) — Your modal editor, which will automatically talk to the `zk` language server for diagnostics, code actions (`space a`), go-to-definition (`gd`), and hover docs (`space k`).

---
### How to Add Them to Your NixOS Config

If you manage your packages declaratively (e.g., in your system configuration or home-manager configuration), you can drop them in like this:

```nix
environment.systemPackages = [
  pkgs.zk
  pkgs.ripgrep
  pkgs.fzf
  pkgs.helix
];

```
---

### Quick 3-Step Setup to Get It Running

1. **Initialize a notebook directory:**
```bash
mkdir -p ~/Notes/zk-vault
cd ~/Notes/zk-vault
zk init

```

*(It will ask you a couple of quick preferences, like whether you want to use WikiLinks—say yes!)*
2. **Configure Helix to hook `zk` into Markdown:**
Inside your notebook folder, `zk` can manage a `.helix/languages.toml` file, or you can add it to your global `~/.config/helix/languages.toml` so it fires up whenever you open a `.md` file inside your notes directory:

```toml
[[language]]
name = "markdown"
language-servers = [ "zk" ]

[language-server.zk]
command = "zk"
args = ["lsp"]

```

3. **Start writing notes:**
* Run `zk new "My First Note"` from your shell, or use `zk edit --interactive` to fuzzy-find and open notes instantly.
* Open Helix on any markdown file, and you'll have full LSP tag/link tracking, `gd` jumping, and code actions out of the box.
---

Would you like to map a quick Fish shell abbreviation (like the ones from the video) so typing `zn` instantly creates a new note in your vault?
___
