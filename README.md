# Stupid

## Types

### Numbers (like)

#### Integers

boolean: 1 or 0, true or false; non fuzzy

> trivia: Church number

byte: 8 bits (why 8? historical reasons)

MSB/LSB: what do you mean you "significant"

we're all live in the contingent shadow of history

word: ALU register size (depending on your architecture)

size_t

nobody is using undetermined size types (`int`, `short`, `long`, etc.) anymore

`cstdint`: `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`

what about signed? zig-zag encoding (used in protobufs)

and for multi-byte integers, endianness.
(spoiler: most of the time you want little-endian, except for network stuff and IBM mainframes)

#### Decimals

##### Fixed point

how do we represent decimals?

a naive approach: fixed point (a forgotten art)

scale by 10/100/1000 etc. (but we have binary computers, not decimal!)

Q4.4 = 4 bits integer, 4 bits fraction = 0..15.9375

LSB has a different meaning: 1/(2^fraction_bits) = precision

(won't discuss arithmetic here, those are implementation details)

##### Floating point

floating point: IEEE 754 standard (single/double precision)

fp32 (float), fp64 (double); fp16, fp8, fp4

weight quantization (1.58bit floats)

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

Variadic length encodings

Unicode, UTF-8, UTF-16, UTF-32

Indeed, we have an problem of efficiency here. (some Chinese keeps arguing about that)
However, we have compression, so let's stop arguing about the encoding and just
use UTF-8

> trivia: emoji, IPA (international phonetic alphabet) and zero-width joiner

### Strings

> There's no string (in C)

This section should have been placed after [Array](#array),
but I just can't resist telling this sad truth earlier.

### Interlude (1)

it should be the end of [primitive types](https://en.wikipedia.org/wiki/C_data_types) (or [fundamental types](https://en.cppreference.com/w/cpp/language/types.html)), except the misplaced [Strings](#strings).

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

### Array

fixed-size (both count of element, also the size of element), contiguous memory block

(except VLA, which just a fancy `alloca`)

A confusing macro:

```c
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]))
```

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

and I refuse the latter, since it's technically an anonymous struct that given a
name via typedef.

Why the `typedef` dance? cuz C struct lives in a different namespace than other types, unlike C++.

Try to guess the size of `a_plex` above.
If the answer is 9 bytes, you're tricked by alignment and padding. 

to make it 9 bytes really, you always could add `__attribute__((packed))`, although the CPU might not happy (and sometimes traps you)

> trivia: bit-field

### Interlude (2)

Memory is flat.

If that’s not obvious, go look at the memory map of a microcontroller like the STM32F411.

In its reference manual, see:

- Chapter 5, Figure 14: Memory map
- Table 10: STM32F411xC/xE register boundary addresses


It's quite hard to explain without introducing Von Neumann architecture, Harvard
architecture, and what a ROM(instruction memory)/RAM(data memory) is.

Nowadays, most of the microcontrollers are (pretending to be) Von Neumann
architecture, which means:

> instructions and data are stored in the same memory space.

now you could look at the memory map again...


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

### Enum

Just a named integer constant. (Go even doesn't have enum)

(atom is way more interesting, but it's not in C)

### Interlude (3)

I said:

> **By itself,** union is just basically useless...

Here comes [Tagged union](https://en.wikipedia.org/wiki/Tagged_union)

### Epilogue

This is the end of C types.
(is it?)

## Assembly
