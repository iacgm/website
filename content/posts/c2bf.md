---
date: '2025-10-01'
draft: false
title: 'C? Rewrite it in Brainfuck'
---

## Doing things worst

People always seem to want to do things well, and when they fail, they tend to blame their tools. So it should come as no surprise that programmers, being somewhat similar to people (and being generally bad at what they do), have a long tradition of growing near-religious zeal for editors, paradigms, code styles, and, of course, programming languages. The bickering never ends, and whatever one person preaches, another [considers harmful](https://en.wikipedia.org/wiki/Considered_harmful).

"_Static typing just gets in the way_", says one voice, "_Python lets me just code._"

"_The borrow checker really isn't **that** bad, once you get used to it_", responds another, whose T-shirt reads "Rewrite it in Rust".

"_Have fun with your loops and mutable state_", snickers a third, "_get back to me when you learn what a monad is._"  

It is just how Boris Marshalov described Congress: "A man gets up to speak and says nothing. Nobody listens—and then everybody disagrees."

But then, miraculously, from the heavens above, we hear the wise, booming voices of [Church and Turing](https://en.wikipedia.org/wiki/Church%E2%80%93Turing_thesis) in unison, bringing clarity through the noise: "It simply does not matter", they say, "When all is said and done, they are all equivalent."

But what if you're not the sort of person who wants to do things well? What if you're the sort of person who wants to things bad? What if you want to do things _worst_?

In that case, you might have something in common with Urban Müller, who in 1993[^Bohm], presumably unable to accept the Church-Turing doctrine, created a language so heinous that no name but "Brainfuck" could suit it. With only 8 instructions and with no support for variables, functions, random access memory, datatypes, fractional arithmetic, memory allocation, or any conventional control flow (!), surely _this_ abomination could not be compared to (say) C, the digital world's lingua franca, used on damn near every device of the last half-century. 

[^Bohm]: With a little help from [Bohm](https://esolangs.org/wiki/P'').

Hello world in Brainfuck, for the uninitiated, reads as follows:

```brainfuck
++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
```

Over the last 30 years, Brainfuck has become a sort of running joke for programmers. They like to point and gawk and say, with mysticism and awe: "_Brainfuck may **look** like a mess, but it's Turing Complete: you can program absolutely anything you want in Brainfuck._" But you can't, because Brainfuck is **hard**, and you're not that smart.

But _**I**_ can.

_**I**_ can make just about anything I want in Brainfuck. For instance, take [this recent Brainfuck creation of mine](/donut.bf), which displays an animation of a spinning donut:

![ASCII Donut](/donut.png#center)

If this looks familiar, it's because it's a (slightly modified) version of Andy Sloane's famous C program, [donut.c](https://www.a1k0n.net/2011/07/20/donut-math.html), translated into Brainfuck. 

No, I did not write all 125,489 instructions of this myself. Instead, I spent 6 weeks writing a [C-to-Brainfuck compiler](https://github.com/iacgm/c2bf), and then modifying donut.c to use extra-low-precision fixed-point arithmetic. 

My compiler supports almost all of C's core features, including:
- integer arithmetic
- local and global variables
- loops & if-statements
- arrays
- functions (& recursion)
- pointers
- function pointers (!)

Here's how I did it.

## Brainfuck Semantics

Before we discuss the compiler iteself, it's worth going over Brainfuck's semantics.  

We start with an infinite tape of cells, each containing the number zero. We have a tape head pointing at the first of these. Then, we have eight instructions, each with its own symbol:
1. `+`: Increment the cell under the tape head.
2. `-`: Decrement the cell under the tape head.
3. `>`: Move the tape head right.
4. `<`: Move the tape head left.
5. `,`: Store the next character from the input into the cell under the tape head.
6. `.`: Output the value stored in the cell under the tape head.
7. `[`: If the tape head points to a zero, jump to the corresponding `]`.
8. `]`: If the tape head does **not** point to a zero, jump to the corresponding `[`.

Essentially, `[`/`]` blocks act as `while-non-zero` loops.

That's it. Really.

## Compiler Primer

This compiler is not really all that different from any other, so is a good basis for learning the basics of compilers. All a compiler does is translate one language into another, with some intermediate steps. This is done in a series of translation passes, each taking us slightly further along the path from the source code to the target language. In my compiler, the stages are:

1. **Lexing**: This step converts a stream of characters into **tokens**. That is, it splits our code into keywords, identifiers, symbols, etc...
2. **Parsing**: This step converts a stream of tokens into an **abstract syntax tree** (AST). That is, it splits our code into constructs: expressions, statements, definitions, etc...
3. **IR Code Generation**: At this step, we convert our AST into an **intermediate representation** (IR): that is, a language that somehow bridges the gap between C and Brainfuck. It should be something easy to translate both into and out of.
4. **BF Code Generation**: At this step, we translate each IR instruction into Brainfuck.
5. **Peephole Optimization**: This step is not really needed, but in order to acheive reasonable execution speeds, we will need to get make our Brainfuck interpreter recognize certain patterns common in Brainfuck which can be sped up, or even skipped entirely.

### Lexing & Parsing

This is, more-or-less, a solved problem. This is the one part of this project where I made use of an external library, [Pest](https://pest.rs/). This allows us to specify our grammar (in this case, the [C99 grammar](https://www.quut.com/c/ANSI-C-grammar-l-1999.html)[^C99]) in something like [Backus-Naur Form](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form). For example:

```
if_stmt = { "if" ~ "(" ~ expr ~ ")" ~ stmt ~ ("else" ~ stmt)? }
```

Once Pest has constructed a parse tree, we can straightforwardly construct our AST. For example, our if-statement struct looks like:

```rust
enum Stmt {
  IfStmt(Expr, Box<Stmt>, Option<Box<Stmt>>),
  // ... every other kind of statement
}
```

The same can be done for other constructs, essentially mindlessly.

[^C99]: I would highly encourage anyone who hasn't done so to take a good, hard look at the C grammar, or at least to peruse the Microsoft language documentation. C is full of quirks that you might otherwise never encounter, and if there's one thing building a compiler forces you to do, it's to really, _really_ learn your language.

### IR Generation

This is really where the magic happens. But before discussing the compilation it's important to consider what the IR will look like. Our IR language must account for the fact that Brainfuck does not have random access memory, and so every decision about where we store each value will create tangible, difficult obstacles when we try to translate into Brainfuck.

This is a bit of a puzzle that I encourage you to consider. Noticing a solution for this problem is what inspired me to work on this project at all.

The idea is to use a [stack-based](https://en.wikipedia.org/wiki/Stack-oriented_programming) IR. What this means is that our instructions will not take any arguments, and we will instead operate on a single stack. This way, when we translate to Brainfuck, we will (for the most part) not have any arguments to retrieve from memory.

For example, consider the statement:

```c
putchar('0' + 3 * 2);
```

Using a register-based IR, for example, would yield something like:

```asm
mov r0, 48     ; 48 is '0' in ASCII
mov r1, 3
mov r2, 2
mul r1, r1, r2 ; multiply r1 & r2
add r0, r0, r1 ; add r1 to r0
putchar r0     ; or whatever the I/O convention is
```

But our stack-based IR has no (non-constant) arguments at all. Instead, each argument acts on the outputs of the previous ones:

```asm
Push(48) ; stack becomes [48]
Push(3)  ; stack becomes [48, 3]
Push(2)  ; stack becomes [48, 3, 2]
Mul      ; stack becoems [48, 6]
Add      ; stack becomes [54]
PutChar  ; stack becomes []
```

Staying with our if-statement example from earlier, the AST node `If(cond, then, else)` would become something like:

```asm
; [IR for `cond`]
Branch(L1, L2) ; Jump to L1 if cond is true, L2 otherwise
Label(L1)
; [IR for `then`]
GoTo(L3)
Label(L2)
; [IR for `else`]
Label(L3)
```

Notice that, while these instructions take arguments, they are all constants. We never have to read any memory apart from the values at the top of the stack.

### Translation into Brainfuck

The stack-based IR is especially nice because Brainfuck's semantics basically provides us with a stack already, in the form of its tape. We can think of the tape head as pointing to the top of the stack and the cells to the head's left as the values on the stack. We should be careful to keep cells to the right of the tape head empty, so that we can use those cells as-needed. With this aproach, translating our first few instructions is a breeze. For example:

1. `Push(5)`: `>+++++`  
We just move the tape head and increment the new cell 5 times.
2. `Add`: `[-<+>]<`  
Slightly more complicated, but not too bad. If `x` & `y` are our arguments, we repeatedly decrement `x` & increment `y`, until `x` is zero. Then, we shift the head back one cell to free a space on the stack.
3. `Store(3)`: `<<<[-]>>>[-<<<+>>>]<`
To move a value from the top of the stack to a slot further down, we just clear that slot, then add the value at the top of the stack, and then move the tape head back.
4. `GoTo`: Hm...
While arithmetic and other basic operations can be implemented by direct translations[^translations], implementing the jumping we need for general control flow (or for function pointers) is more complicated.

[^translations]: "Direct" does not mean "simple"; just try implementing bitwise operations without multiplication or division. Even unsigned comparisons are highly nontrivial. Take a look at [these](https://esolangs.org/wiki/Brainfuck_algorithms) for inspiration.


Again, I encourage you to stop and think through this. This is another puzzle.

This time, the idea is to use a whole-program transformation. We can wrap the whole program in a `[`/`]` loop, and then gate each block of assembly in an if-statement. This way, we only need to implement one simple kind of control flow in Brainfuck. As an example:

```c
a:
  // Block A
  goto c;
b:
  // Block B
  exit;
c:
  // Block C
  goto b;
```
... can be rewritten as ...
```c
int label = 'a';
while (label != 0) {
  if (label == 'a') {
    // Block A
    label = 'c';
  }
  if (label == 'b') {
    // Block B
    label = 0;
  }
  if (label == 'c') {
    // Block C
    label = 'b';
  }
}
```

Equality checks, while-non-zero loops, and if-statements are all, if not easy, at least tractable problems to address in BF itself.

As for pointer-reads-and-writes, I will make no attempt to explain these, beyond linking [this excellent article](https://www.inshame.com/2008/02/efficient-brainfuck-tables.html).

I've skipped over all sorts of caveats and minutiae, for example, the details of our calling convention and how we access global variables from inside function calls. This has been to everyone's benefit.

### Peephole optimization

While the above steps does let us compile working Brainfuck programs, they aren't close to optimal. Just looking at a snippet of our donut.bf code, we see all sorts of suboptimal code:

```brainfuck
+<[[-]>-<]>[-<+>]<[-<[-]<>>+>+++++++++++++++[-<[->>+>+<<<]>>[-<<
+>>]>[-<<<+>>>]<<]<<<[->>>+>+<<<<]>>>>[-<<<<+>>>>]<>>[-]+>[-]+>[
-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>
[-]+>[-]<[<]<[->>[>]<[--[++++[->]>]++<]>--<<[<]<]<[->+<]>>>[>]+>
[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+
>[-]+>[-]+>[-]<[<]<[->>[>]<[--[++++[->]>]++<]>--<<[<]<]>>[>]++++
++++++++++++[-<[-<<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>>]>[-<+>]<]<[<]
<+>>[>]<[>+<----[[-]>-<]>[-<<[<]<[-<+>>+<]>[-<+>]>[>]>]<<[<]<[->
++<]>[-<+>]>>[>]<]<<<<[-]>[-<+>]<[->+>+<<]>>[-<<+>>]<>++++++++++
```

All over the place, we see sequences like `<>` or `><`, which can be eliminated entirely. Even worse, we also find slow  _but optimal_ code, such as long strings of `>`/`<` & `+`/`-` instructions, `[-]` sections, and "move" blocks (i.e., blocks which clear a cell and add or subtract its value from other cells, such as `[-<+>]` or `[->>+>+<<<]`).

These are not hard to detect, and doing so allows us to skip-ahead and execute large blocks at a time in one fell swoop. Doing so speeds up our code massively. For example, adding 100 to 100 would normally take about 600 instructions, but instead becomes a single instruction.

I tried rendering a single frame of donut.c without these optimizations, but gave up after about 20 hours becaues my laptop started to overheat so much that my screen became unresponsive. Some back-of-the-napkin math shows I could have expected to wait about 28 hours on my machine.

With these optimizations, we can render a frame in about 90 minutes.

We can take this even further though, because some profiling reveals that only a small number of complex instructions take up almost all of our runtime. After using a similar trick to detect complex instructions like bitwise operations, multiplications, divisions, and memory accesses, I can render a frame in about 12 minutes.[^caveat] 

[^caveat]: In order to be able to do this safely, we have to be sure that whenever we see the code for a bitwise operation (for example), it truly does just compute value, with no side effects. This means we have to be sure that all memory cells that that snippet of code could read have the values we expect. We can do this by having these instructions clear all the memory they need before using any of it.

Yes, it's kind of cheating. No, I don't really care.

## Limitations

There are some features missing, but for the most part, they would not be hard to add. The limiting factor was just that around this point, this project went from fun and educational to time-consuming and tedious.

Some major missing features are:
- Support for variadic functions (including `printf`) and library functions more generally.
- Differently-sized types (apart from arrays). Most Brainfuck interpreters use 1-byte cells. Mine uses 2-byte cells so they are large enough to store the labels to be jumped to, and so that I could have enough precision to implement usable fixed-point arithmetic.
- Switch statements. No excuse here, just got a bit lazy.
- Memory allocation / variable length arrays. This would be a complicated addition to make, since we would have to change how we access global variables, but it would not be impossible.

## Similar Projects

After completing this project and doing some research, I found that the idea for this project, like any good idea, has been had several times before. However, while I definitely commend anyone who undertakes building a X-to-Brainfuck compiler, I do think my approach sets this project apart from the others.

Several projects (such as the one described [here](https://www.bozidarevic.com/2019/12/transpiling-c-into-brainfuck/)) exist which can translate simple commands, and even loops, but do not support things like function calls. The linked article even goes so far as to describe the `goto`-convention I used, but says it'd be too difficult to implement.

The only ones I've found which are as complete as mine are ones which do work, but are more like emulators written in Brainfuck than transpilers. That is, rather than translate the code directly, they act like virtual machines, putting instructions into memory and then executing each one independently, simulating traditional memory and registers. [ELVM](https://github.com/shinh/elvm) is a project which targets many esolangs, but the BF implementation is essentially an emulator. Gregor Richards has [another project](https://esolangs.org/wiki/C2BF) which is explicitly an emulator.

There's nothing wrong with emulators, but I feel my approach gives more of a "true" translation (whatever that means), and the stack-based IR really is the secret ingredient for that. It's probably also much faster, since we just need to actually execute each instruction, as opposed to loading instructions from memory and dealing with program counters and registers and so on. I didn't benchmark them though, because fundamentally, what difference does it make. So it goes.
