# Scratch 10-26-26

## Should I change the `o` for my `Delete` key

- Change it to `O` because when i hold space and turn `h,j,k,l` into
`Left,Down,Up,Right, Arrows` and hold `LShift` its natural to just then press
the `o` for `O` instead of having to make sure i release shift first?
- Doesnt matter, `Delete` and `LShift + Delete` is still `Delete`.

## Org Cheats + Fish Auto Syntax and Helix Markdown Templates

Haha, right?! Absolute madness! Nobody in their right mind sits there manually
typing out `# + b e g i n _ s r c` character by character every single time
they want to paste a three-line shell snippet.
If people had to do that manually, the entire Emacs/Org community would have
collective carpal tunnel!

That's the beauty of `org-tempo`—it brings the exact same single-symbol speed of
Markdown into Org mode. You just type `<s` + `Tab` and boom, the whole block expands
around your cursor instantly.

### Quick Cheatsheet for Org-Tempo (`<` + key + `Tab`)

| Type this... | Hit `Tab` and it expands to... |
| --- | --- |
| **`<s`** | `#+begin_src ... #+end_src` (Source Code Block) |
| **`<e`** | `#+begin_example ... #+end_example` (Raw Text/Output) |
| **`<q`** | `#+begin_quote ... #+end_quote` (Blockquote) |
| **`<v`** | `#+begin_verse ... #+end_verse` (Verse / Poetry / Song) |
| **`<c`** | `#+begin_center ... #+end_center` (Centered Text) |
| **`<l`** | `#+begin_export latex ... #+end_export` (Raw LaTeX Export) |

---

### How to Lock It In Your Config

Just make sure this chunk is in your `~/NixOS/Master/Modules/Features/doom/doom.d/config.el`:

```elisp
;; Enable classic Org tempo completion (<s TAB, <e TAB, etc.)
(after! org
  (require 'org-tempo)
  
  ;; Custom tempo templates for instant language blocks
  ;; Example: type <n TAB for nix, <el TAB for elisp
  (add-to-list 'org-structure-template-alist '("n" . "src nix"))
  (add-to-list 'org-structure-template-alist '("el" . "src elisp"))
  (add-to-list 'org-structure-template-alist '("sh" . "src bash")))

```

Now you don't even have to type `nix` or `elisp` after the block opens!
You just type *`<n`* + *`Tab`* and it creates a full `#+begin_src nix` block with
your cursor waiting right inside it. Zero reaching, zero `#+` keys,
100% home row efficiency!

## Textbook Zettelkasten Process

That is a **textbook Zettelkasten / inbox-processing workflow**, and it is by
far the most sustainable way to take notes without killing your momentum.

By separating *Capture* (frictionless brain dumps during the day) from
*Processing* (organizing, filing, and tagging later), you get the best of both worlds.

---

### The Capture vs. Processing Pipeline

[ Quick Draft ] --> [ Single Append Scratch ] --> [ Batch Processing Script ]
──> [ Atomic Org Notes ]
(During the day) --> (Scratch-YYYY-MM-DD.md) --> (Next morning / end of day)
--> (Permanent Vault)

1. *During the Day (Zero Friction):* You keep one rolling file open
(e.g., `Scratch-2026-10-26.md`).
You drop thoughts under `###` headings, write your semantic lines, and close it.
No naming, no tagging, no filing.
2. *End of Day / Next Morning (Batch Processing):* You run a quick processing
sweep to turn those heading blocks into individual atomic files inside your
permanent Org vault.

---

### The Fastest Way to Process

Instead of manually creating a file, copying text, pasting it, adding headers,
and deleting the section, you can automate 90% of the mechanical work.

#### Option A: Fish Script (`refine-scratch`)

You can write a simple command line utility in Fish using `pandoc` or `awk` that
takes your daily scratch file, scans for every `###` or `##` heading, and
automatically splits each section into its own `.org` file using the heading
name as the filename and `#+title:`.

For example, a heading like:
`### Org Cheats and Fish Auto Syntax`

Gets automatically generated as:
`~/Org/Nodes/org_cheats_and_fish_auto_syntax.org`
with pre-populated Org headers:

```org
#+title: Org Cheats and Fish Auto Syntax
#+date: 2026-10-26
#+filetags: :inbox:

<pasted content block>

```

Then, all you have to do is open Emacs, hit `SPC n f` (or your note searcher),
jump into the newly generated node, tweak the tags, and you're done.

#### Option B: Native Doom Emacs / Org-Roam Refile (`C-c C-w` or `SPC m r r`)

If you convert the whole scratch file into a single temp `.org` file first via `pandoc`:

1. Open `Scratch-2026-10-26.org` in Doom Emacs.
2. Put your cursor on a heading.
3. Use Org-roam's built-in extraction command: *`M-x org-roam-extract-subtree`*
(or bind it to a single key).
4. Emacs will automatically take that entire heading block, cut it out of the
scratch file, prompt you for tags, and save it as its own clean, standalone atomic
Org node!

---

### Recommended Daily Routine

1. **Start Day:** Fire up `Scratch-YYYY-MM-DD.md` in Helix.
2. **Throughout Day:** Dump everything inside without worrying about structure.
3. **End of Day / Start of Next:**

- Convert or extract the blocks into individual Org nodes
(via script or `org-roam-extract-subtree`).
- Add your tags during the quick review.
- Archive or delete yesterday's scratch pad.

This keeps your mind completely clear while drafting, while making sure your
final Org vault stays structured, searchable, and pristine.

## Doom Interactive Subtree File Extraction

Both approaches complement each other perfectly: the Fish function lets you rapidly
process files straight from the terminal, while `org-roam-extract-subtree` gives
you a surgical, interactive hotkey inside Doom Emacs when you're working directly
inside an Org buffer.

Here is how to set up both.

---

### 1. Doom Emacs: Interactive Subtree Extraction Keybinding

`org-roam-extract-subtree` is built into Org-roam. It takes the heading under your
cursor, removes it from the current buffer, and creates a brand-new atomic Org-roam
file out of it using the heading text as the title.

Add this binding to your `~/NixOS/Master/Modules/Features/doom/doom.d/config.el`:

```elisp
;; Bind org-roam-extract-subtree for instant atomic node creation
(map! :leader
      :prefix ("n" . "notes")
      :desc "Extract subtree to node" "x" #'org-roam-extract-subtree)

```

#### Workflow in Emacs

1. Open your converted daily scratch file in Doom Emacs.
2. Place your cursor anywhere on a `*` or `**` heading.
3. Press **`SPC n x`**.
4. Org-roam creates the atomic file, prompts for tags/aliases if needed, saves it
to your node directory, and deletes the extracted block from your scratch file.

---

### 2. Terminal: `refine-scratch` Fish Function

If you want to automate the batch conversion straight from your command line without
opening Emacs first, you can use a clean Fish function.

Create `~/.config/fish/functions/refine-scratch.fish`:

```fish
function refine-scratch --description "Auto-split markdown scratch headings into
atomic Org nodes"
    set -l input_file $argv[1]
    set -l target_dir ~/Org/Nodes # Adjust to your actual Org nodes directory path

    if not test -f "$input_file"
        echo "Error: File '$input_file' not found."
        return 1
    end

    # Ensure target directory exists
    mkdir -p $target_dir

    # First, convert markdown to temporary org buffer via pandoc
    set -l temp_org (mktemp --suffix=.org)
    pandoc -f markdown -t org "$input_file" -o $temp_org

    # Process headings and extract into separate .org files
    awk -v out_dir="$target_dir" -v date_str=(date +%Y-%m-%d) '
    BEGIN { current_file = ""; }
    /^(\*|\*\*|\*\*\*) / {
        # Clean up heading title for filename
        raw_title = $0;
        sub(/^(\*|\*\*|\*\*\*) /, "", raw_title);
        
        filename = tolower(raw_title);
        gsub(/[^a-z0-9_-]/, "_", filename);
        gsub(/_+/, "_", filename);
        sub(/^_/, "", filename);
        sub(/_$/, "", filename);
        
        if (filename == "") next;

        current_file = out_dir "/" date_str "_" filename ".org";

        # Write Org Frontmatter
        print "#+title: " raw_title > current_file;
        print "#+date: " date_str >> current_file;
        print "#+filetags: :inbox:\n" >> current_file;
        next;
    }
    {
        if (current_file != "") {
            print $0 >> current_file;
        }
    }
    ' $temp_org

    # Clean up temp file
    rm $temp_org

    echo "Successfully split '$input_file' into atomic Org nodes in '$target_dir'."
end
```

#### Workflow in Terminal

1. Run `refine-scratch Scratch-2026-10-25.md`.

2. Every `###` section instantly becomes a standalone `.org` file inside
`~/Org/Nodes/` (formatted with `#+title:`, `#+date:`, and tagged `:inbox:` for later
review).
3. Once processed, you can delete or archive the scratch file and open a blank scratch
pad for the new day.

Would you like to test running `refine-scratch` on your current `Scratch-10-25.md`
file to verify how the output `.org` nodes land in your vault?

## Org-Mode & Org-Roam Core Features

That strategy is bulletproof. Separating *capture* (raw, frictionless dumping
in the scratchpad) from *processing* (metadata, properties, graph linkage) is
the only way to build a massive knowledge vault without getting bogged down in
endless refactoring loops.

Since you're using *Org-mode* and *Org-roam* as the engine behind this
database, understanding its native features will allow you to structure everything
cleanly during that processing step so every note is automatically
indexable, queryable, and connected.

---

### Org-mode Core Features (The Structure Layer)

Org-mode isn't just text; it's a structured tree format with
properties, execution capabilities, and metadata attached to every node.

#### Document Metadata & Headlines

Every Org file and headline can act as an independent data node.

- *Headlines (`*`, `**`, `***`):* Define hierarchical folding nodes.
- *Keywords (`TODO`, `DONE`, `RESEARCH`, `HOLD`):* Native state tracking for action
items embedded right inside your documentation.
- *Tags (`:rust:std:memory:`):* Fast, non-hierarchical categorization. Tags inherit
down headline trees.

#### Property Drawers (`:PROPERTIES:`)

Property drawers are hidden key-value stores attached to any specific headline or
file. This is where Org transforms from flat text into a *queryable database*.

```org
* Option<T> & Result<T> Enum Memory Layout
:PROPERTIES:
:ID:       eb7a23c4-8f12-4c22-b919-94bf8c3da872
:CATEGORY: rust-std
:TYPE:     type-layout
:COMPILER_VERSION: 1.82.0
:SOURCE:   https://doc.rust-lang.org/std/option/
:END:

```

- *Why it matters:* You can use tools like Org-Agenda or custom Elisp script functions
to query your vault for specific properties
(e.g., "Find all notes where `:TYPE:` is `type-layout` and `:CATEGORY:` is `rust-std`").

#### Literate Programming & Babel Code Blocks (`#+BEGIN_SRC`)

In Org-mode, code blocks aren't just for syntax highlighting—they can be executed
directly inside the buffer using *Org-Babel*.

```org
#+BEGIN_SRC rust :exec cargo run :results output
fn main() {
    println!("Size of Option<u8>: {} bytes", std::mem::size_of::<Option<u8>>());
    println!("Size of Option<NonZeroU8>: {} bytes", std::mem::size_of::<Option<std::num::NonZeroU8>>());
}
#+END_SRC

```

- *Why it matters:* As you research the Rust standard library, you can write
micro-benchmarks or pointer-checking snippets directly inside your research notes,
hit `C-c C-c`, run `cargo` behind the scenes, and print the output directly below
the code block.

---

### Org-Roam Features (The Graph & Linkage Layer)

Org-roam sits on top of Org-mode and manages an automated *SQLite database* indexing
all your files, headlines, links, and property drawers in real time.

#### Node Creation (`org-roam-node-find` / `org-roam-node-insert`)

- *File Nodes:* An entire `.org` file representing one overarching topic
(e.g., `20261026054900-rust_std_allocator.org`).
- *Headline Nodes:* Any headline with an `:ID:` property drawer automatically
becomes an independent node in the Org-roam database without needing its own
separate file.

#### Aliases & Roam Tags

- *Aliases (`#+ROAM_ALIASES:` or `:ROAM_ALIASES:`):* Assign multiple names to
a single concept node so you can link to it naturally under different terms.
- *Example:* A node titled `Ownership & Borrowing` can have aliases
`"Borrow Checker"`, `"Lifetimes"`, and `"Move Semantics"`. Inserting a link to any
of those terms links to the exact same core node.

- *Roam Tags (`#+FILETAGS:`):* High-level categorization used by Org-roam to filter
search completion buffers (e.g., `#+FILETAGS: :rust:concept:`).

#### Direct Links vs. Backlinks (Graph Dynamics)

- *Direct Links (`id:...`):* Point from the current note directly to a specific
target node or headline.
- *Backlinks:* Org-roam automatically tracks every node that links *to* the node
you are currently viewing. You never have to manually construct cross references
when you open a note on `std::pin::Pin`, the Org-roam sidebuffer shows every other
note in your system that references `Pin`.

---

### Optimal 2-Step Workflow Pipeline

┌─────────────────────────────────────────┐
│     Step 1: Scratchpad / Buffer         │
│  - Raw capture during research          │
│  - LSP / Linter cleans syntax           │
│  - Zero organization stress             │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│     Step 2: Processing & Indexing       │
│  - Move into Org / Org-roam Vault       │
│  - Assign ROAM_ALIASES & FILETAGS       │
│  - Link to parent concepts (id:...)     │
│  - Add Babel code execution blocks      │
└─────────────────────────────────────────┘

By strictly keeping Step 1 about *pure capture* and Step 2 about *database properties
and graph linking*, you completely eliminate wasted time spent re-editing existing
notes later. Once a note passes Step 2, it is permanently locked into your graph
database, fully searchable, and instant to retrieve.

Would you like to map out the exact Org-roam capture template configuration for
Doom Emacs (`org-roam-capture-templates`) so that Step 2 automatically prompts you
for aliases, tags, and parent links?
