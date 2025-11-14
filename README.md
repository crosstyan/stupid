# Stupid

## Types

### Numbers (like)

#### Integers (a lie agreed upon)

Computers don’t have integers.

They have bits. We’re the ones who insist that a certain pattern of bits "is a
number", and not, say, a color, a character, or the high score of some dead
arcade machine.

We start with the simplest lie:

`bool` — one bit. 1 or 0. True or false. Non-fuzzy, allegedly.

Of course, the compiler will happily store it in a whole byte anyway, because
hardware likes alignment more than it likes philosophical minimalism.

> trivia: Church number

`byte` — 8 bits. 

Why 8? Historical reasons. We could have standardised on 9. Or 12. We didn't.
Now all your text, file formats, and protocols silently assume
"8" like it was handed down from the mountain.

**MSB / LSB**: "Most Significant" and "Least Significant" bit.  
What do you mean, "significant"? It’s just the side we decided to treat as "the
big end."  

> we all live in the contingent shadow of history

Then we get to the mess: `int`, `short`, `long`.

Once upon a time, these meant something like "the machine's natural *word* size".

By "word", it has a vaguely similar flavour to the *Western-centric* idea of a "word":
imagine a single letter is a bit, and a word is a meaningful unit made from multiple
letters (bits). It's not a correct analogy (you'd need information theory to really
do it justice), but that’s roughly how I imagine the term "word" stuck.

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

GB2312, Big5, Shift-JIS, EUC-JP

Unicode, UTF-8, UTF-16, UTF-32

Indeed, we have an problem of efficiency here. (some Chinese keeps arguing about
that) However, we have compression, so let's stop arguing about the encoding and
just use UTF-8

> trivia: emoji, IPA (international phonetic alphabet) and zero-width joiner

### Strings

> There's no string (in C)

This section should have been placed after [Array](#array),
but I just can't resist telling this sad truth earlier.

### Interlude (1)

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

### Interlude (2)

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

to make it 9 bytes really, you always could add `__attribute__((packed))`, although the CPU might be not happy (and sometimes traps you)

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

### Interlude (3)

I said:

> **By itself,** union is just basically useless...

Here comes [Tagged union](https://en.wikipedia.org/wiki/Tagged_union)

Remember [struct](#struct)'s other name? **Product type**.

Now we have **Sum type**.

### Epilogue

This is the end of C types.
(is it?)

If you're the one who is familar with C, you might have noticed that there's something I (intentionally) missed.

## Assembly

go back to von Neumann architecture again.
