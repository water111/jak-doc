# Why GOAL?
Naughty Dog developed the four Jak games in GOAL, an entirely custom LISP-based programming language for the Playstation 2.  Developing a programming language is a big task, and only makes sense if there's a huge benefit to doing so.  What were these benefits?

## Reduced Edit/Compile/Link/Test Cycle Time
In my opinion, this is the single greatest benefit of the GOAL language.  PlayStation 2 games were often written in C or C++, so the process of making a change and seeing the result required:
- Save the file
- Compile changed files and link the main executable
- Reset the Playstation 2 and load the executable
- Wait for the game to load
- Move the player to the spot for testing

With GOAL, functions and variables could be replaced interactively at runtime. You could either type in the code you wanted to run at a prompt, or select some text in your editor and use a keyboard shortcut to trigger a "compile, upload, link, execute" cycle. When developing, the programmer could upload code which redefined a function or changed the value of a global variable. The process of "compile changes, send to game, changes are applied" happens in a fraction of a second, and allows programmers to see the effect of their changes immediately. 

User-defined data types also had the ability to `inspect` themselves, which simply meant printing out the name and value of each field in the type.  Users could define their own `print` and `inspect` methods for types to customize this as needed, but the defaults were used in almost every place.  When combined with the ability to patch in a `printf` to any function as the game was running, this allowed programmers to be extremely efficient.

## Easy Inline Assembly
This is another killer feature of GOAL - the ability to easily use inline assembly. GOAL allowed programmers to mix together high-level and low-level programming in the same function, all with a similar syntax.  Special register variables could be declared, which could be used directly with both assembly and high-level code.  These variables also worked with the type system.  Optionally, the compiler could take care of register allocation for you, and GOAL's register allocation was quite good.  Most of the PS2's power was locked behind custom instructions added by Sony to a standard MIPS III CPU, so using inline assembly was required to have a high-performaing engine.  Inline assembly is used all over the place in the Jak games.

Another key feature to make this work well is function inlining, which allows snippets of high-perfomance assembly to easily be inserted without having to worry about the low-level details.  For example, there is a `abs` function, is always inlined. Even though the function is always inlined, the compiler still generates a standalone function so we can see how it was implemented
```
    or v0, a0, r0
    bltzl v0, L302
    dsubu v0, r0, v0

L302:
    jr ra
    daddu sp, sp, r0
```
I know this isn't "standard GOAL" because the compiler will never emit a `bltzl` opcode. Once this clever function was written, it can be easily used anywhere, without thinking about the low-level implementation details. 

The combination of function inlining and inline assembly working with the register allocator means that these inlined functions will play well with the other code in the function, and can often eliminate unnecessary moves (if the argument to an inlined `min` is not needed after the `min`, the result of the `min` will often end up in the same register as one of the arguments, for example).  Here's an example of `abs` in the wild:
```
    mfc1 v1, f0
    bltzl v1, L283
    dsubu v1, r0, v1
L283:
```
notice it's using different registers, and the `or` at the beginning is eliminated and the register coloring system is able to move the value straight from `f0` to `v1` without passing through a temporary.

Trivia: `vector+!` and `vector-!` were not inlined by default in the Jak 1 demo, but are inlined in all versions after.


## Data Layout
GOAL had powerful tools for creating complicated types. Programmers could manually specify offsets of fields, and could easily create types where fields overlapped in a certain way.  This is all possible in C, but requires confusing nesting of `struct`s and `union`s, and error-prone manual bookkeeping to insert a bunch of padding fields.  

GOAL also had bitfield types that were similarly customizable and supported efficient reading/writing of bitfields up to 128-bit. This is not that much of a technical achievement, but C/C++ managed to do bitfields unbelievably poorly (and `gcc` couldn't do 128-bit bitfields), so it is worth mentioning. 

GOAL also had good tooling for large static data, like textures, geometry/DMA chains, VU microprograms, and entire level scene graphs, which could then be easily included as static data available to the programmers.  Sony's PS2 SDK had the ability to do some of these things, but often required a terrible custom language/assembler to generate an object file, or generating huge text files that were then processed by the normal C compiler.  Jak 2 and later games also had the ability to read/write data from a SQL database at runtime, and I suspect that Jak 1 could at least read these databases at compile time to automatically generate arrays of config data.

## Macros / Customization
The GOAL language has a powerful Scheme-like macro language which can transform code at compile time.  When combined with an inline assembly system that can understanding branching and works with register allocation, it allows you to use macros to define new syntax, like a while loop. 

For even more powerful customization, programmers could modify the compiler directly!  Some examples of where this used:

- The cooperative multi-tasking "process" system. Each game "actor" (including things like collectables) had its own relocatable "process" memory and a small area (default 256 bytes!) to back up a stack ("thread").  Implementing memory access into the process memory required addressing relative to a special `s6` "process register", which could only be done with the cooperation of the compiler.

- Virtual-inheritance state system for actors.  I'd like to document this system more in the future, but each process can define "state"s which seem to be a list of handler functions which run on state transitions, once per frame, etc. Types can inherit / override their list of `state`s like they can with methods.

- Ability to link new code/data in at runtime.

- Integration with other tools to import data.

## Higher Level Language Features
GOAL has a number of higher-level language features that most PS2 games didn't really use, like runtime type information (optional), efficient boxed integers, single-inheritance type system, lambdas (kinda, not really sure if variables were "captured" correctly), and virutal method system. The language was developed in 1999 (I think...), and does pretty well when compared to `gcc` for MIPS from the same period in terms of performance.


# The Downsides of GOAL.
Using a custom language also has some downsides.  These are mostly just guesses, considering that there is not much information out there about how GOAL was bad.

## Lack of Optimizations
With just a single person writing a compiler, it's impossible to get performance that's as good as a well-established compiler, with hundreds of person-years of development time.  While GOAL code wasn't terribly inefficient, and did allow developers to easily write high performance assembly when needed, plain GOAL code was worse than `gcc` code in a number of ways.  

The GOAL compiler was pretty clearly a single pass compiler.  This means that some common optimizations like "don't calculate an unused value" or "common subexpression elimiation" can't be implemented. Also there is no optimization for moving instructions to delay slots (though there were some "pre-canned" control flows that did do something useful in them), or reordering to get more dual issue.  

 There are also some situations where the compiler generates less than optimal code - the easiest to spot example being that floating point constants were stored as static data that had to be loaded from memory. As a result, a function which was probably written as `(defun 1/ ((x float)) (/ 1. x))` turned into
 ```
    daddiu sp, sp, -16  ;; allocate space on stack for fp
    sd fp, 8(sp)        ;; backup old fp
    or fp, t9, r0       ;; set fp to address of this function
    lwc1 f0, L345(fp)   ;; load 1.0
    mtc1 f1, a0         ;; move arg to fpr
    div.s f0, f0, f1    ;; divide!
    mfc1 v0, f0         ;; move result to return reg
    ld fp, 8(sp)        ;; restore fp
    jr ra               ;; return/reset stack
    daddiu sp, sp, 16
``` 
 
 The `lwc1` causes a significant number of cache misses in some otherwise very efficient inner loops.  Note that Jak 2 gets this right and loads into a gpr using a `lui`/`ori` pair, then moves the data to an fpr. The same function, in Jak 2:
 ```
lui f0, XXXX
ori f0, f0, XXXX
mtc1 f1, a0
div.s f0, f0, f1
mfc1 v0, f0
jr ra
daddu sp, sp, r0
``` 
  
  This way probably easy to find with the Performance Analyzer TOOL, but it wasn't available until after Jak 1's release, so they only had the CPU's performance counters available to assess performance.

There are also a lot of cache misses around function calls, as each function call requires loading the function address out of the symbol table, then jumping to this address:
  ```
    lw t9, name=(s7)
    lw a0, -2(gp)
    or a1, s5, r0
    jalr ra, t9
    sll v0, ra, 0
```
  
  The symbol table is 16 kB, so there's a good chance that a random access to it can cause a cache miss.  This indirection is required for the interactive code patching to work, as changing a function is accomplished by updating the symbol table to point to the new function. But you could imagine a "release" build where all the function calls to always-loaded functions use a normal direct jump and link (`jal`) instruction, instead of a `lw`, `jalr` combo. This would remove a cache miss on a huge number of function calls. 

A final example of a missed optimization is the size of all the symbol names, which are stored in RAM so that new code can be linked at runtime.  In a release build, you could have saved around 0.5 MB of RAM by omitting these strings and using some unique ID instead. This was eventually implemented in Jak 3.

Despite this, Jak 1 is a seriously efficient engine that runs at a rock solid 60 fps. (Due to the way it handles interlacing and only buffering one frame at a time in VRAM, if 60 fps fails, it must run at 30 fps to avoid halving the vertical resolution, so it's obvious when it drops to 30 fps)  Take a look at this performance analyzer snapshot of a single frame (cut off when the CPU is done)

![Image](frame1.png?raw=true "Image")

There are some low-performance areas without much dual issue and and with many cache misses (likely creature code, character control code, etc), but a huge portion of the frame time is spent in a super-optimized handwritten assembly.  The Jak 1 renderers used a _ton_ of CPU. Especially the environment mapping renderer.  Later games shifted more of the work to VU1 as the CPU was too busy with doing all the collision and other stuff required to make the people walk around the city.



## Missing Features
GOAL is missing a few features, likely due to the complexity of implementing them.  GOAL has good support for 128-bit integers, meaning you can use them as function arguments and return values, and that the compiler knows how to load/store them, and move them around in registers, and can do register allocations with them.  There's a (basically unused?) bitfield type that shoves 4 floats into a 128-bit general purpose register, but this is not that useful, as the 128-bit GPRs do not support any floating point operations. 

However, the PS2 also has VU0 macro mode vector registers, which are also 128-bit, but work with the VU0 macro mode floating point vector operations. Unlike the 128-bit GPRs, the GOAL compiler doesn't play well with these VU0 registers.  The only real way to use them is with inline assembly.  This means that any reference to a floating point vector in the "non-inline-assembly" part of the language actually has to be a pointer to 128-bits in memory containing a vector.  So a "vector of 4 floats" object is really not a "128-bit value containing 4 floats", but instead a "reference to memory containing 4 floats".  This adds some extra loads/stores, and means that vectors end up on the stack a lot. Also, inlining these functions is not as efficient. For example, the `vector+!` function:

```
    vmove.w vf6, vf0
    lqc2 vf4, 0(a1)
    lqc2 vf5, 0(a2)
    vadd.xyz vf6, vf4, vf5
    sqc2 vf6, 0(a0)
    or v0, a0, r0
    jr ra
    daddu sp, sp, r0
``` 
 
 Whenever the `vector+!` function above is inlined, it always contains those loads and stores, and always uses those exact same registers, because the compiler doesn't really know how to manipulate these vectors stored in `vf` register.  The `vector+!` function also blows away whatever is in the `vf4`, `vf5` and `vf6` registers, and the compiler doesn't know this, as it doesn't seem to do coloring for `vf` registers.  It also ends up always using `a0`, `a1`, and `a2` exactly like in the inline assembly - perhaps coloring doesn't happen at all on VU macro mode assembly instructions?
 
As a result, very efficient use of the VU0 macro mode instructions is only possible when the entire routine is written with inline assembly. 



## Cost/Frustration to Develop a Language
My understanding is that the GOAL compiler was written entirely by Andy Gavin and took about a year.  This meant he didn't contribute much (at all?) to Crash Team Racing.  While GOAL may have been worth it in the end, I am sure it was frustrating at the time to have one of the very few programmers spending their full time developing a tool instead of directly working on the engine.

There's also some evidence that GOAL had some small issues during development.  In particular, they used ~7600 out of 8192 symbols.  This is probably the most they could use before the performance of the hash table became very poor.  As a result, they probably had to worry about things like "you can't add another global variable or named function because we are out of symbols", which was probably frustrating.

Trivia: The exact size of the symbol table is mysterious.  

- There's a `printf` that lists the max number of symbols as 8192, which would require 65536 bytes to store.
-The MIPS instructions would allow it to be at most 8192 symbols (each 8 bytes) and still access all symbol values with a single `lw` instruction.
- The "symbol info" table is stored at an offset of 65336 bytes away from the start.  Observant readers will note that 65336 is slightly smaller than __65536__, could this a typo?
- Even more observant readers will note that 65336 / 8 = 8167 symbols, which is a prime number! It's a common practice to use a prime number for a hash table size so that `value % hash_table_size` can provide some extra "shuffling" if your hash function is bad.  In this case `value` is the `crc32` hash of a string.  But the benefits of this "prime number trick" are totally lost because the function used is effectively `value % 8192`, and values which hash to over the size of the table effectively hash to zero. I don't think this was the intention because 8191 is a better prime number to pick than 8167.
- The actual implementation ends the symbol table at 65280 (`0xff00`), which is yet another number.  Maybe it crashed when the table ended at `0x10000`, and the size was just decreased until it started working?


(This was "improved" in Jak 2 by doubling the number of symbols, and some other system I don't entirely understand for level types, but they still got very close to the limit!)

