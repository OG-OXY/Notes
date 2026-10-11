Exactly—you’ve hit on the distinction between a **map/dictionary** and a **list/array**.

* **Key-Value Pairs (No Dash):** When a key points to a single scalar value (like a boolean, string, number, or path), you just write `key: value`. For example, `inputs: /literal/path/to/dir` or `enabled: true`. It’s just a property belonging to that block.
* **Lists (With Dashes):** You only use a dash `-` when a key represents a collection that can contain *multiple* items, where order or multiplicity matters (like a list of rules, packages, or keys).

So if a setting only takes one value, it doesn't need a bullet point. It only needs a dash when it's part of a multi-item collection!

[[THE WAY]]