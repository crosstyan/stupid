# Stupid

## Intro

This documentation is intended to be read **with** a search engine and a Large Language Model (LLM) open next to you.

No prior knowledge is needed, only curiosity and patience.

> ...this is a lie.
>
> I'm secretly expecting you to have touched at least one of:
> C / C++ / Rust / Go / JavaScript / Python.
>
> But if you’re curious enough, and you’re allowed to Google / ping an LLM,
> you’ll probably find your way through anyway.


What this documentation is **not** :

* a C tutorial
* an x86/ARM reference
* an OS textbook
* a functional programming textbook
* a step-by-step "learn to code" guide

I will happily skip "obvious" syntax, wave my hands over some details, and tell you:

> Ask the LLM: *explain this thing in 5 lines*, you've seen the context.

You are expected to:

* look things up,
* try tiny experiments,
* run your own code,
* and occasionally tell the LLM "no, that's wrong, try again".

Roughly I only cover:

- the *models*
- the *connections*
- the *taste* (at least my taste)

You should imagine this as the *spine*, and the rest of the internet as *flesh* you can grow on demand.

---

If you're still here after this introduction, congratulations: you are probably the target audience.

Turn the page. We start with bits, and some lies about integers.

## Types

### Numbers (like)

#### Integers (a lie agreed upon)

Computers don't have integers.

They have bits. We’re the ones who insist that a certain pattern of bits "is a
number", and not, say, a color, a character, or the high score of some dead
arcade machine.

We start with the simplest lie:

`bool` — one bit. 1 or 0. True or false. Non-fuzzy, allegedly.

Of course, the compiler will happily store it in a whole byte anyway, because
hardware likes alignment more than it likes philosophical minimalism.

> trivia: Church number

`byte` — 8 bits. 

Why 8? Historical reasons. We could have standardized on 9. Or 12. We didn't.
Now all your text, file formats, and protocols silently assume
"8" like it was handed down from the mountain.

> we all live in the contingent shadow of history

Then we get to the mess: `int`, `short`, `long`.

Once upon a time, these meant something like "the machine's natural *word* size".

> [!NOTE]
> By "word", it has a vaguely similar flavour to the *Western-centric* idea of a "word":
> imagine a single letter is a bit, and a word is a meaningful unit made from multiple
> letters (bits). It's not a correct analogy (you'd need information theory to really
> do it justice), but that’s roughly how I imagine the term "word" stuck.

That made sense when machines were small and weird. Today we have `<cstdint>`
because nobody wants to play "guess the number of bits" anymore. We summon
`uint32_t` and it does what it says on the tin. Mostly.

Signedness is its own little horror story.

`int32_t` vs `uint32_t` vs "some encoding that pretends to be both". ZigZag
encoding, in protobuf, is basically a hack that says: "what if we pretended
signed integers were just unsigned integers, but we wrapped them in a clever
zig-zag pattern so small negative numbers compress well?

The machine doesn't care. The ALU just sees 32 bits.
We're the ones saying "this is an age" or "this is −1" or "this is a file
descriptor". Types aren't in the silicon; they live in the agreement (in human sense).

and for multi-byte integers, endianness.
(spoiler: most of the time you want little-endian, except for network stuff and IBM mainframes)

##### More on endianness

The problem of endianness comes from human languages/writing system.

Most of the sane world writes left-to-right (like English, *modern* Chinese,
etc), like how we write numbers: the most significant digit on the left, the
least significant digit on the right, so called MSD (most significant digit) first,

i.e.

```text
1234 = 1*10^3 + 2*10^2 + 3*10^1 + 4*10^0
```

In right-to-left scripts (Arabic, Hebrew), text flows right -> left, but the digits
themselves are still written in the same order we use: the "1" in 1234 is still
the thousands place, the "4" is still ones. So you get this weird hybrid: text
goes one way, numbers another.

In a hypothetical LSD (least significant digit) first writing system, the same numbers
would be written like this:

```text
4321 = 4*10^0 + 3*10^1 + 2*10^2 + 1*10^3
```

while the same representation `1234` would mean:

```text
1234 = 1*10^0 + 2*10^1 + 3*10^2 + 4*10^3
```

> [!NOTE] 
> Don't forget top-to-bottom ("column-first") writing systems (traditional
Chinese/Japanese/Korean). If you treat the page as a 2D array, you’re
essentially:
>
> - going down for each character in a column
> - then moving left to the next column
>
> We don't normally write numbers bottom-to-top (thankfully), but the idea of
> "which direction do we advance the position in?" is the thing that matters.

Back to computers.

Due to the flat memory model, and a given address grows direction, how would you
interpret a multi-byte integer stored in consecutive memory addresses?

```txt
consider a uint32_t stored at address 'a':

0A0B0C0D (MSB first, big-endian, like how we write/read numbers)
    = 0x0A * 2^(8*3) + 0x0B * 2^(8*2) + 0x0C * 2^(8*1) + 0x0D * 2^(8*0)

low addresses  -->  high addresses
|a |a+1|a+2|a+3|
|0A|0B |0C |0D |  # big-endian
|0D|0C |0B |0A |  # little-endian
```

> [!IMPORTANT] MSB/LSB
> "Most Significant" and "Least Significant" bit.
>
> "Significance" here means "how much weight this position carries *as a number*":
> the MSB contributes the largest power of 2, the LSB the smallest.
>
> Fortunately, nobody uses a "middle-endian" writing system where the most
> significant digit lives in the middle... as far as I know. Consider it an
> exercise to design such a monstrosity.

---

We also have the problem of bit endianness (I'd call it significance problem).

with the writing direction analogy:

```txt
low addresses --> high addresses
------------------------------
|hello|world|
|world|hello| # endianness problem
|oehll|lrwod| # significance problem
```

Here, "endianness problem" means you swapped whole words: `hello world` → `world
hello`.  "Significance problem" means you scrambled the characters *inside* each
word, as if the bits in a byte were being read in the opposite order.

*note that, if you changes the low/high address direction, the endianness would be flipped too*

Usually the significance won't be a problem (any sane system would do MSB first inside a byte)
However, the problem of significance mostly relate to bitfields. i.e.

```c
struct bit_field_plex {
    uint8_t a:3;
    uint8_t b:5;
};
```

which comes first, `a` or `b`? it depends on the compiler implementation.
(but *usually* it's LSB as first field, and for each number of field, MSB first inside the field)

> [!WARNING]
> by usually I mean most of the little endian common compilers implementations, your mileage may vary

You could verify what your compiler does with this code:

```cpp
#include <cstdint>
#include <utility> // for std::unreachable in C++23

constexpr bool is_lsb_first_in_struct() {
    const struct test {
        uint8_t all_ones: 4;
        uint8_t all_zeros: 4;
    } t = { .all_ones = 0b1111, .all_zeros = 0b0000 };
    static_assert(sizeof(t) == 1, "unexpected size");
    if (const auto p = reinterpret_cast<uint8_t*>(&t); *p == 0b11110000) {
        return false; // msb first
    } else if (*p == 0b00001111) {
        return true; // lsb first
    } else {
        // shouldn't happen in sane system
        std::unreachable();
    }
}
```

#### Decimals

How do we represent decimals?

##### Fixed point

Naive (and underrated) approach: **fixed point** — a forgotten art.

Idea: scale by 10 / 100 / 1000 etc., store as integer.  
Problem: we have binary machines, not decimal.

So we usually do binary fixed-point: pick a Q-format like **Q4.4**:

- 4 bits integer, 4 bits fraction → range `0 .. 15.9375`
- The LSB of fraction part means `1 / 2^fraction_bits` → here, `1/16 = 0.0625` precision.

(won't discuss arithmetic here, those are implementation details)

##### Floating point

floating point: IEEE 754 standard (single/double precision)

fp32 (float), fp64 (double); fp16, fp8, fp4 (latest NVIDIA GPUs support them)

> trivia: neural networks and quantization

needs FPU, which is a separate register set

### Characters

#### ASCII

7 bits, 128 characters, 1 byte aligned
non-printable control characters (0-31)

have you ever seen `^C` in terminal? that's a `0x03`
(the non-printable is actually printable)

#### Rune

a beautiful word borrowed from Go language, which doesn't exist in C, but
worth mentioning

National encoding:

- GB2312
- Big5
- Shift-JIS
- EUC-JP

International encoding (Unicode):

- UTF-8
- UTF-16
- UTF-32

Indeed, we have an problem of efficiency here and some Chinese keeps arguing
about that. However, we have compression, so let's stop arguing about the
encoding and just use UTF-8

> trivia: emoji, IPA (international phonetic alphabet) and zero-width joiner

### Strings

> There's no string (in C)

This section should have been placed after [Array](#array-1),
but I just can't resist telling this sad truth earlier.

### Interlude (end of primitive types)

it should be the end of [primitive types](https://en.wikipedia.org/wiki/C_data_types) (or [fundamental types](https://en.cppreference.com/w/cpp/language/types.html)), except the misplaced [Strings](#strings).

### Array (1)

Our first **generic type** -- although historically it was treated as something more
"primitive" than generics. Languages had arrays long before they had parametric
polymorphism.

Now that we do have generics, we have to retro-fit arrays into that world. I
like to call this hole punching: C's array is basically a hole in its type
system.

Let’s use C++ syntax instead of C, because it makes the intent explicit:

```cpp
std::array<T, N>
```

Where:

- T = element type (must have a known size)
- N = number of elements (must be a compile-time integer)

This immediately raises the important question:

Why must the size of T be known?

Hold that thought.

### Interlude (memory is flat)

> Memory is flat.

If that’s not obvious, go look at the memory map of a microcontroller like the STM32F411.

In its reference manual, see:

- Chapter 5, Figure 14: Memory map
- Table 10: STM32F411xC/xE register boundary addresses


It's quite hard to explain without introducing Von Neumann architecture, Harvard
architecture, and what a instruction memory (what we call ROM/Flash) and data memory (what we call RAM) are.

> trivia: there's also IO to handle; CPU is just a dumb calculator, things
unrelated to calculation/control flow are the realm of peripherals

Nowadays, most of the microcontrollers are (at least pretending to be) Von Neumann
architecture, which means:

> instructions and data are stored in the same memory space.

What's my question again? "Why a known size is so important?"

> memory is flat

go back to array again

### Array (2)

> An array is a data object holding elements of the same type, identified by a
numeric index. Elements are allocated consecutively in memory.
> 
> -- [GNU C Language Manual](https://www.gnu.org/software/c-intro-and-ref/manual/html_node/Arrays.html)

except VLA (variable-length array), which just a fancy `alloca`

---

In flat memory, the whole point of an array is that you can find element `i` by:

> `address_of(arr[i]) = base_address_of(arr) + i * sizeof(T)`

This only works if `sizeof(T)` is known and fixed at compile time.
If you don’t know the size, you can't compute the offset.


a famous macro:

```c
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]))
```

### C-String

null-terminated byte array; or called sentinel-terminated array

```c
char str[] = "hello";
```

equivalent to

```c
char str[/* 6 (optional, the compiler could infer the size) */] =
    {'h','e','l','l','o','\0'};
```

equivalent to

```c
char str[6] = {0x68, 0x65, 0x6c, 0x6c, 0x6f, 0x00};
```

Did I said a single quote `'` is for a single character?
(I'm not teaching you C syntax here, just a reminder)

My opinion is here: it's a bad design, please go for slice/span/view, if you
have a choice (often you don't, sadly)


### Slice/Span/View

someone said it's fat. (it is fat)

to proper write it down, you would need have to know [`struct`](#struct) first.

spoiler alert:

```c
struct slice_t {
    void* ptr;
    size_t len;
};
```

Please don't ask what `void*` really is.
In this context, I genuinely have no idea either.
(And I promise that's not a joke.)

### Struct

Our second generic type (after array)

> Ross sat on the Algol 68 committee with C.A.R.Hoare in the mid-1960s,
where his previous work on a record-like data structure (called a **plex**)
influenced Hoare's own ideas on abstract data types...
>
> -- Alan Blackwell and Kerry Rodden (2003) 
> Sketchpad: A man-machine graphical communication system reissued edition with
preface by Blackwell and Rodden

See [Casey Muratori – The Big OOPs: Anatomy of a Thirty-five-year Mistake – BSC 2025](https://www.youtube.com/watch?v=wo84LFzx5nI) for more of this story

Long story short: we probably should have called it a plex instead of struct.
(It sounds cooler, and is historically closer.)

Well, it has other names: record, tuple (field name is replaced with position
index), product type (in type theory).

> trivia: algebraic data types (why product?) 

Anyway, here's a struct

```c
struct a_plex {
    uint32_t a;
    float b;
    int8_t c;
};
```

You then have to write `struct a_plex` everywhere (in C), which gets old fast.

so many people do this:

```c
typedef struct a_plex a_plex_t;
```

Also, many people like to do this:

```c
typedef struct {
    uint32_t a;
    float b;
    int8_t c;
} a_plex_t;
```

I refuse the latter, because technically that's:

- an anonymous struct
- that’s been given a name only via `typedef`.

It works, but the struct itself is never get registerd in the struct namespace.
I like my structs (or plexes) named.

---

Why the `typedef` dance? 

cuz C struct lives in a different namespace than other types, unlike C++.

---

Try to guess the size of `a_plex` above.
If the answer is 9 bytes, you're tricked by alignment and padding. 

to make it 9 bytes really, you always could add `__attribute__((packed))`,
although the CPU might be not happy (and sometimes traps you)

> trivia: bit-field


### Union

The size of union is the size of its largest member.

However, it never knows which member is currently active.
That's not the job of the union itself.

**By itself,** union is just basically useless, but here's a trick I'd like to share:

```c
struct color {
    union {
        struct {
            uint8_t r;
            uint8_t g;
            uint8_t b;
            uint8_t a;
        };
        uint32_t rgba;
        uint8_t components[4];
    };
};
```

Reinterpret the same memory block in different ways.

### Enumeration

Enum

Just a named integer constant and Go even doesn't have enum.

[atom](https://www.erlang.org/doc/system/data_types.html#atom) is way more interesting, but it's not in C.

### Interlude (tagged union)

I said:

> **By itself,** union is just basically useless...

Here comes [Tagged union](https://en.wikipedia.org/wiki/Tagged_union)

Remember [struct](#struct)'s other name? **Product type**.

Now we have **Sum type**.

### Epilogue

This is the end of C types.
(is it?)

If you're the one who is familiar with C, you might have noticed that there's something I (intentionally) missed.

## Assembly

Go back to Turing machines / von Neumann architecture and
forget the high-level control flow: `function` / `if` / `while` / `for` / `switch` / etc.
Those are also illusions.

In a sense, today's computers are not much different from punched-tape machines:
a head, a tape, a position, and some rules about how to move.

Before reading this section, go to [Introduction](#introduction-assembly) and
try to **think like a CPU**.  
(And stay aware of your *position* in the text.)

> You shouldn’t read this line.

### Branch

Branching = changing the `PC` (program counter) register to a different address.

In prose form:  

> if some condition, jump to *there* instead of continuing *here*.

You've just done a branch: the text told you to go to another label
([Introduction](#introduction-assembly)) instead of reading straight down.

If you're reading this paragraph *before* reading Introduction, you've mis-executed
the program. Please go back to [Introduction](#introduction-assembly) and try again.

*(Yes, I know this is duplicated if you followed the instructions properly.
Welcome to unstructured control flow.)*

### Introduction (Assembly)

Registers: small boxes inside the CPU. To do a calculation, you first have to put values into boxes.

Instruction Set Architecture (ISA): the set of instructions the CPU understands.
Or, if you prefer: the minimal operations the hardware can perform.

`PC` register (Program Counter): the address of the **next instruction** to execute.
If no branch happens:

- fetch instruction at `PC`
- execute it
- increment `PC`
- repeat

If a branch happens:

- set `PC` to some other address
- execution continues there instead

Does anybody mention branching?

- If you **don’t** know what that means yet, go to [Branch](#branch).
- If you **do**, you are now allowed to keep reading past this point.

### Interlude (memory input/output)

When we say the word *register*, we might mean a quite different thing

- Flip-flop/latch, who remember bits (do they really "remember"/"memorize"? or
are we anthropomorphizing circuits?)
- the small boxes inside the CPU, where we do calculations (ALU's)
- thouse indicate the state of CPU (like PC, SP, etc) they have nothing to do
with calculation
- some special memory addresses, when we read/write to them, it's not really
memory access (although we still treat them as memory, as von Neumann's magic
suggests), but actually interact with peripherals (or, parts that are not part
of CPU core, not involving control flow or calculation)
- some field in protocol like Modbus, where specific the abstraction index of
the target of operation (you can't still do much except read/write in syntax
sense, but anything could happen, including trigger a nuclear missile launch)

All of these get casually called "register" in various contexts.
Do these match a rigorous CS definition of "register"? Not really.
Let’s ask a dictionary what it thinks:

> a device (as in a computer) for storing small amounts of data
especially : one in which data can be both stored and operated on (Merriam-Webster)

> A mechanical indicating device or apparatus; esp. one which automatically
records data or measurements. (Oxford English Dictionary)

> In an adding machine or a calculator (mechanical or electronic): a device or
system for displaying or storing the results of arithmetical operations or other
numerical data. (Oxford English Dictionary)

> Computing. A temporary memory location able to store only a single string of
bits (typically equal in size to the maximum word length allowed by a computer's
architecture), but having a high access speed; (in later use) esp. one of a set
of such locations in a central processing unit. (Oxford English Dictionary)

So, to positively define "register" is fuzzy and context-dependent.
But we can at least say what a register is not:

> memory in broder sense, nontheless the data or instruction memory, reguardless the medium:
RAM, ROM, Flash, disk, tape, punch card, etc.

To make things more confusing, von Neumann architecture encourages us to *pretend*:

> instructions, data, and peripheral registers all live in **one flat address space**,
> and we use the *same load/store instructions* to touch all of them.

(That's true on e.g. ARM Cortex-M with memory-mapped I/O. On x86, there's also a
separate `in`/`out` I/O space, just to keep life spicy.)

So you do:

```c
uint32_t *p = (uint32_t *)0x40021000;
uint32_t x = *p;      // looks like "just read memory"
*p = x | 0x01;        // looks like "just write memory"
```

and what actually happens is:

* you just turned on a GPIO port,
* or started an ADC conversion,
* or acknowledged an interrupt,
* or something equally non-RAM-ish.

> welcome to the confusing world of computer architecture

If you want a concrete example, open the STM32F411 reference manual and look at:
Table 10. STM32F411xC/xE register boundary addresses

Try to tell, just from the map:

- which region is instruction memory (Flash),
- which region is data memory (SRAM),
- which regions are peripheral I/O registers.

Those peripheral "registers" are not RAM. They only pretend to be memory
locations so the CPU can talk to them with the same load/store machinery.

They're just doors in the flat address space, leading outside the nice clean
world of "bits in memory" into motors, LEDs, UARTs, radios, and other rude
realities.

And it's not the only way to do it: remember the Modbus example?  
You'll meet similar "registers" in I2C, SPI, CAN, etc.

In all of these, the "register" you read/write is just an **addressed slot in a protocol**.  
What actually happens behind that slot is completely up to the device:

- maybe it really is a flip-flop
- maybe it's a FIFO
- maybe it's a write-only command trigger
- maybe it quietly arms a missile launch (hopefully not)

The pointer, the address, the "register number"... is just the **signifier**.

The signified is *void* until you bring in a specific datasheet, device, and context.
(If you like Lacan: the signifier is empty; the meaning is not in the token, but in the
network of relationships around it.)

### Control Flow

You only need `if` and `goto`. 

Indeed, [Go To Statement Considered Harmful](https://wiki.c2.com/?GotoConsideredHarmful).
In structured programming sense, Dijkstra might be right. However the "structure" itself
is still arbitrary. (why `while` remains, but not `until`?)

CPU doesn't care, it only knows:

- sequence: "do this, then that" (and the PC just increments)
- unconditional jump: `goto <label>` -> set PC to label address
- conditional jump: `if <condition> goto <label>`

Those are the "primitives". 

- sequence just happens naturally
- `B` (ARM) / `JMP` (x86): unconditional jump
- `BEQ` / `JZ`: branch if equal / jump if zero; `CBNZ`/`CBZ`: compare and branch
if not zero/zero (ARM)

// some example to desugar `while`/`for`/`switch` into `if` + `goto`

---

A defend of using `goto` in **C**:
C just such a incomplete/lack of feature of a language;

If C has (in my personal priority order): 

1. closures (lambdas) with proper capture semantics
2. `RAII` (Resource Acquisition Is Initialization)
3. `defer` statement
4. `try`/`catch`/`finally` so called *exception*

then we could finally say: 

> `goto` is dead for good.

without those, error handling without `goto`, only with in the structured
programming primitive of C is a nightmare.

> At the machine level, it's all `goto` and conditional branches anyway.

and by the way: I like early returns. (but what's a `return`?)

### Call Convention

> a `return` is just a special kind of `goto` to an address saved by the calling
convention, plus some stack fixing.

incomplete

### Call Stack

incomplete

a nudge to the implication of "Everything is continuation"

### Pointer

incomplete (relate to the missing part of C types: Chekhov's gun goes fire)

the return of the missing `void*`

Who's the pointee? boring answer: *it depends*.
(but remember: we only have some registers, and a flat memory space)

---

Assume we have two registers:

* `R1` holds some integer value (what we call an immediate value)
* `R2` holds a **memory address**

(I'm using arbitrary register names; I don't care about each ISA's actual naming scheme here.)

Conceptually, in C++-ish pseudocode, think:

```cpp
int32_t R1;
int32_t* R2;

static_assert(sizeof(R1) == 4);
static_assert(sizeof(R2) == 4); // assuming a 32-bit architecture
```

(remember: types are just agreements about how to interpret bits; R1 and R2 are
both small boxes of 32 bits inside the CPU, assuming a 32-bit architecture)

Now, take this operation:

```cpp
R1 = R1 + *R2;
```

Read as:

> Load the 32-bit value at address `R2`, add it to `R1`, store the result back
into `R1`.

On x86-ish assembly, that’s exactly:

```asm
add R1, [R2]    ; R1 = R1 + *(int32_t*)R2
```

On ARM (A32) it would be two instructions, because ARM separates load and ALU ops:

```asm
LDR  R3, [R2]   ; R3 = *(int32_t*)R2
ADD  R1, R1, R3 ; R1 = R1 + R3
```

On RISC-V (RV32):

```asm
lw   t0, 0(R2)  # t0 = *(int32_t*)R2
add  R1, R1, t0 # R1 = R1 + t0
```

Same story: one register used as a **pointer**, a load, then an add.

---

Now the opposite direction: writing back through the pointer.

In C++-ish form:

```cpp
*R2 = *R2 + R1;
```

Read as:

> Load the 32-bit value at address `R2`, add `R1`, write it back to the same address.

x86-ish:

```asm
add [R2], R1    ; *(int32_t*)R2 = *(int32_t*)R2 + R1
```

ARM:

```asm
LDR  R3, [R2]   ; R3 = *(int32_t*)R2
ADD  R3, R3, R1 ; R3 = R3 + R1
STR  R3, [R2]   ; *(int32_t*)R2 = R3
```

RISC-V:

```asm
lw   t0, 0(R2)  # t0 = *(int32_t*)R2
add  t0, t0, R1 # t0 = t0 + R1
sw   t0, 0(R2)  # *(int32_t*)R2 = t0
```

---

The important part isn't the exact opcode spelling; it's the pattern:

* **Pointer** = "a register whose bits we choose to interpret as an address"
* `*ptr` in C/C++ = "issue a load/store instruction using that register as the base address"

So when we say in C:

```cpp
extern int32_t some_arbitary_number;

int32_t *p = &some_arbitary_number;
int32_t x = *p;
*p = x + 1;
```

On a RISC-y ISA, the CPU sees something morally equivalent to:

```asm
# assume p is in a register, say a0

lw   t0, 0(a0)    # t0 = *(int32_t*)p
add  t0, t0, 1    # t0 = t0 + 1
sw   t0, 0(a0)    # *(int32_t*)p = t0
```

You see, the pointer isn’t some high-level magic: it’s just the language exposing the
**primitive load/store pattern** the ISA already has.

You also see that high-level languages mostly avoid talking about **specific numeric addresses** directly:

- you name a variable (`some_arbitrary_number`),
- the compiler/linker decide *where* it lives,
- you use `&` to get its address, and `*` to read/write through it.

The only common exceptions are things like **memory-mapped I/O registers**, which must live at fixed addresses (because the hardware says so).

You *can* still control variable placement manually:

- via linker scripts,
- attributes (`__attribute__((section("...")))`, etc.),
- storage class specifiers in C/C++ (`static`, `extern`, etc.),

but in ordinary code it’s usually discouraged or just unnecessary.

Most of the time, you let the toolchain pick addresses, and you think in terms of:

- **names** (symbols, variables)
- and **relationships** (pointers between them),

while the CPU only ever sees:

- registers full of bits,
- and load/store instructions using those bits as addresses.

---

What makes pointers so hard to grasp in C is that C **mixes multiple concepts into one syntax**.

Let’s borrow Zig’s pointer model to clarify what’s going on:

* **single item reference**: `*T`

  > "This points to exactly one `T`."
* **many items, unknown length**: `[*]T`

  > "This points to the first element of some contiguous `T`s. I don’t know how many."
* **sentinel-terminated**: `[*:0]T`

  > "Many items, and you promise there’s a special terminator value (`0` here) at the end."
  > This is what a C string really is: `[*:0]u8` a.k.a. `char*` with `\0` at the end.
* **slice**: `[]T`

  > “A fat pointer: `{ ptr: *T, len: usize }`.
  > I know *where* it starts and *how many* elements there are.”

These are **different ideas**:

* "exactly one thing"
* "an array I can walk but don’t know the length"
* "an array that ends when I hit a sentinel"
* "an array with an explicit length"

In C, they are all just fucking `T*`.

* pointer to single `int`? — `int*`
* first element of an array of `int`? — `int*`
* pointer to a C-string (null-terminated `char` array)? — `char*`
* beginning of a memory-mapped register block? — `uint32_t*`
* slice/view/span? — usually `T*` + `size_t` separately

The language does **not** encode:

* whether there's 1 element or many,
* whether it's safe to index `p[10]`,
* whether there's a terminator,
* whether you're allowed to write to it,
* whether it's even “normal memory” or MMIO.

You get one syntax (`T*`) trying to wear all those hats at once.

That's why beginners (and honestly, plenty of non-beginners) get lost:

the **machine model** is simple:

  > here is an address, do a load/store

but the **semantic model** (what this pointer *means*) is carried purely in:

  * comments,
  * conventions,
  * and whatever lives in the programmer’s head.

Zig's pointer types are basically just the language admitting:

> These are different shapes of 'pointer', let's name them separately.

### Symbol

incomplete

mangling/function overloading/linking

and what `static` really means (in both linkage and storage lifetime sense)

### See also (Assembly)

- [Learn X in Y minutes: Where X=MIPS Assembly](https://learnxinyminutes.com/mips/)
- [A friendly introduction to assembly for high-level programmers — Hello](https://shikaan.github.io/assembly/x86/guide/2024/09/08/x86-64-introduction-hello.html)
- [CONDITIONAL EXECUTION](https://azeria-labs.com/arm-conditional-execution-and-branching-part-6/)
- [Branch instructions](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/The-Instruction-Sets/Branch-instructions)

## Lisp

incomplete: not really about Lisp the language, which is boring

(a deceiving chapter title, I know, but can't think of a better one)

> Python is the Lisp less cool

Let's see what a different set of primitives could give us.

the idea of interpreter, *without registers or a flat memory model*.

So called "declearative programming" is actually built on a *set of very
different primitives* from "imperative programming".
(If you get so used to FP that you can't feel the fact, just to see how [Prolog](https://en.wikipedia.org/wiki/Prolog) works and tell me how you feel.)

REPL (and Jupyter/nREPL), shell and GUI

the React idea: UI just a function of state

map/filter/reduce

interlude: typeclasses/concepts/protocols/interfaces/traits: ad-hoc polymorphism

![Haskell Numbers](https://pic.blog.plover.com/prog/haskell/numbers/haskell-numbers-1.1.svg)

interlude: array programming, and there's a language called APL/BQN/J/K
(but Matlab & NumPy & Julia are the same idea in disguise)

epilogue: What's a monad? (plant a seed, not fully explain, just list some examples)
(List, Optional/Maybe, Result/Either)

(note to self: how to pack these loose ideas together?)


## Time

incomplete

> welcome to the desert of the real

CPU frequency, and what a time base really is

first construct of non von Neumann architecture: interruption

Timer as the simplest peripheral (from 555 to SysTick)
*why it's necessary? CPU doesn't know time by itself*
(and pre-emptive multitasking needs time, that's another story)

push/pull model (and the duality of it)

## Scheduler

why `delay` is bad (in `Arduino` sense)
*blocking is dangerous/inefficient*

naive round-robin loop (event loop)

what's being round-robined: state machine

coroutine as state machine

(stackless)

---

another way: 

manipulation of stack (context switch)

(stackful)

pre-emptive or not? that's a question

## Everything is continuation

incomplete

function pointer -> closure -> (coroutine/generator/thread/callstack/exception) -> continuation

// better idea?
