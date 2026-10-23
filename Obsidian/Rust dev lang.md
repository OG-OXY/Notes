Rust and enterprise ecosystems like .NET target completely different tiers of software engineering, meaning your choice dictates both your daily problem-solving style and how you get paid.

Here is the direct comparison of what they mean for your career, your wallet, and your growth as a developer:

## The Core Difference

|Aspect|Rust (Systems Engineering)|.NET / C# (Enterprise & Web)|
|---|---|---|
|What You Build|Operating systems, window managers, game engines, infrastructure tooling, high-throughput crypto/finance backends.|Scalable corporate platforms, cloud APIs, microservices, internal business logic tools, desktop software.|
|The Hard Problem|Memory safety without a garbage collector, strict concurrency boundaries, raw CPU and pointer optimization.|Managing vast business logic, data persistence layers, scalable cloud architecture, third-party system integrations.|
|Market Style|Smaller volume of jobs, but exceptionally high skill ceiling and premium compensation for seniors.|High volume of jobs worldwide. Consistent demand, stable career tracks, and lucrative contract market (like your friend’s).|

---

## What is Better for Pay?

- The Floor (.NET): .NET has a much higher density of jobs across the globe. Getting your foot in the door is easier, and mid-to-senior engineers easily pull six figures. If you go the specialized contract route (like migrating old Windows infrastructures to modern `.NET 10` on Linux microservices), the payouts can be massive because enterprise companies have deep pockets to fix critical blockages.
- The Ceiling (Rust): Rust jobs are fewer, but because the language requires a deeper grasp of memory layouts and hardware, the compensation skews higher on average. Companies hiring for Rust are usually tackling bleeding-edge problems (Web3/Crypto, FinTech high-frequency trading, massive scale backend rewriting at places like AWS or Cloudflare). They pay top-of-market rates because the density of developers who can survive Rust systems programming is much smaller.

---

## What Will Increase Your Capability Most?

Writing a window manager from scratch in Rust is a trial by fire that will force you to become an elite developer incredibly fast.

- Why it maximizes capability: You will have to interface directly with low-level X11 or Wayland protocols, manage window state trees, coordinate asynchronous input events, handle graphics server drawing, and do it all while satisfying Rust’s strict borrow checker. You aren't just writing "code"—you are learning how an operating system coordinates with hardware and memory.
- The Catch: It is a massive structural leap for a first project. You will spend days getting stomped by compiler errors, but if you push through it, enterprise programming in languages like C# or Java will feel trivial by comparison.

---

## Diagnosing Your Shell Glitch (`devenv.nix` + `flake.nix`)

Before you can even write your first line of Rust, we need to fix why your terminal is breaking when you `cd` into your project root.

Since you are using Ghostty and Fish shell, the issue is almost certainly a conflict between how `direnv` / `devenv` hooks into your shell environment and how Fish handles paths. When `devenv` or `direnv` evaluates a flake on folder entry, it attempts to manipulate environment variables. If it outputs standard bash syntax into Fish, your shell will glitch out or crash.

Let's fix it or bypass it completely so you can start coding:

## Option 1: The Fast Bypass (Isolate the Dev Shell)

Don't let automated hooks break your interactive session. Turn off any local directory auto-loading, step into the directory, and run the shell manually using standard Nix flags:

```fish
nix develop --impure
```

_(The `--impure` flag is often necessary if `devenv` is looking at local system state or uncommitted files)._

## Option 2: Check the Fish Direnv Hook

If you are using `direnv` to auto-load your `devenv.nix`, make sure you are using the native fish hook in your `~/.config/fish/config.fish`:

```fish
direnv hook fish | source
```

If your terminal glitches immediately upon `cd`, try temporarily moving `.envrc` out of the directory (`mv .envrc .envrc.bak`) and see if the glitch stops.

To truly understand how memory, scheduling, CPU caches, and the kernel interact, you need resources that bridge the gap between abstract computer science and raw systems engineering. Since your goal is to build a window manager from scratch, you want material that focuses on how Linux manages hardware and processes, as well as how to write high-performance code that doesn't trigger CPU cache misses.

Here are the best, highest-utility resources to master these concepts:

## 1. The Definitive Books (Systems & Kernel Foundations)

- "Operating Systems: Three Easy Pieces" (OSTEP) by Remzi H. Arpaci-Dusseau and Andrea C. Arpaci-Dusseau.
    
    - _Why it’s perfect:_ It is completely free online and widely considered the most accessible, punchy breakdown of CPU scheduling, virtual memory translation, and concurrency ever written.
    
- "Linux Kernel Development" by Robert Love.
    
    - _Why it’s perfect:_ This is the gold standard for understanding how the Linux kernel specifically implements process scheduling (the Completely Fair Scheduler), virtual memory management, and interrupt handling.
    
- "What Every Programmer Should Know About Memory" by Ulrich Drepper.
    
    - _Why it’s perfect:_ A legendary, free, deep-dive paper. It explains exactly how CPU caches (L1, L2, L3) work, how data flows from RAM to the processor registers, and how to write cache-friendly data structures.
    

## 2. High-Performance Rust Resources

- "The Rust Systems Programming Course" (and blogs by Jon Gjengset)
    
    - _Why it’s perfect:_ Jon's YouTube channel and resources deep-dive into complex Rust topics like async runtimes, raw pointers, and atomics.
    
- "The Rust Performance Book" (Official online guide)
    
    - _Why it’s perfect:_ It contains concrete sections on minimizing cache misses, optimizing heap vs. stack allocations, and structuring your data layouts so the CPU can pre-fetch it efficiently.
    

## 3. Window Manager Specific Engineering

- The Linux DRM/KMS and Wayland/X11 Protocols
    
    - To get a window manager on screen, you'll need to learn how the kernel handles graphics. Look into documentation for Direct Rendering Manager (DRM) and Kernel Mode Setting (KMS), which let your code talk directly to your GPU. [1]
    - Reading the source repositories of modern, highly optimized Linux compositors (like Hyprland or smithay) will show you exactly how these concepts look in production code. [1]
    

---

It is awesome that you're taking the dive into Rust systems programming! Laying down those first lines of code on a custom window manager is going to be incredibly rewarding.

Before you can start writing code, do you want to fix that terminal glitch you're running into when you `cd` into your project directory, or would you like a minimal Rust X11 connection snippet to save for your first file?

  

[1] [https://mail.google.com](https://mail.google.com/mail/?extsrc=sync&client=h&plid=ACUX6DN2Vw5BahjH8p1VJDAnC8CDLZ9aNJYouOU)