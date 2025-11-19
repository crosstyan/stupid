## Time

and so on we haven't introduced Time yet (the very thing floating in the air)

> welcome to the desert of the real

So far everything has been suspiciously timeless.

* instructions execute
* control flow jumps
* interpreters evaluate
* UI updates

…but **at what pace**?
Against **what clock**?
Relative to **which outside world**?

We’ve been talking inside a sealed symbolic system. Time is where the outside leaks in.

### CPU frequency and “what a time base really is”

A CPU doesn’t “know” what a second is.

It only knows:

* there is some **clock source** (crystal, RC oscillator, PLL…)
* it gets **ticks** at some rate (e.g. 168 MHz)

Humans call that “168 MHz”.
The CPU just experiences:

> tick, tick, tick, tick

Each tick lets it advance some part of its internal pipeline, but the CPU has no innate sense of:

> “10 ms have passed”

Without extra help, all it can say is:

> “I have consumed N cycles since reset.”

A **time base** is the thing that turns that into something like “milliseconds”:

* a **counter** that increments on each tick (or divided tick)
* a way to **observe** it or compare it against a target

That’s all a timer is, really: a machine that converts raw ticks into “say when”.

> say when

### Timer: the simplest peripheral

From a 555 timer on a breadboard to SysTick on a Cortex-M, the pattern is the same:

```text
[clock] -> [divider] -> [counter] -> [compare] -> [output]
```

* clock: something oscillates
* counter: counts up (or down)
* compare: “have we reached N yet?”
* output: a flag, a pin toggle, an interrupt request (IRQ)

Why is this necessary?

Because:

* CPU cycles are **internal**: “I executed some instructions.”
* Time is **external**: “the world moved on while you were doing that.”

If you want to:

* blink an LED at 1 Hz,
* sample a sensor every 10 ms,
* time out a network packet,
* pre-empt tasks fairly,

you need something that can look at the ticks and say:

> “Now. This is when.”

The CPU cannot feel that on its own. It only feels **changes in signals** (e.g. an IRQ line going high).

### First non–von Neumann construct: interruption

Our von Neumann fairy tale so far:

1. fetch instruction at PC
2. decode
3. execute
4. increment PC
5. repeat

All control flow came from *your* jumps and branches.
Your code owned the continuation.

Now we wire a timer to the interrupt controller, and suddenly:

* when the counter hits a value, it asserts an **interrupt request**
* the CPU:

  * saves some context,
  * loads a new PC from an **interrupt vector table**,
  * jumps to your interrupt handler

Your nice linear story gets a crack:

> you are in the middle of doing something
>
> and then *something else* happens that you did not directly ask for.

From the CPU’s point of view, an interrupt is:

* an **external demand**: “drop what you’re doing and run this other continuation”
* that does **not** fit into your previous flow of `if` / `while` / `call`

This is the first thing in the book that truly doesn’t fit into the “single instruction stream under your control” model.

If you like Lacan:

* your code, your PC, your stack = **Symbolic** order (“this, then that”)
* the timer interrupt = a little piece of the **Real** cutting in:

  * an event whose cause lies outside your current narrative

### Push / Pull (and their duality)

Now that the world can *poke* you, there are two ways to write programs:

#### Pull model

> “I will check when I’m ready.”

Examples:

* polling a GPIO in a loop
* `while (1) { if (uart_has_data()) read(); }`
* checking `millis()` in Arduino to see if enough time passed

Control over “when to look” stays with you.

#### Push model

> “The world tells me when something happened.”

Examples:

* timer interrupt fires every 1 ms
* UART RX interrupt on each received byte
* GUI event loop delivering “mouse moved”, “button clicked”

Here, **the world** (hardware, OS, framework) pushes events into your code.

They’re dual:

* you can emulate push with pull:

  * main loop polls flags set by ISRs
* you can emulate pull with push:

  * event loop calls your function every frame even if nothing changed

But at the hardware level, timers + interrupts are deeply push-shaped:

> Your code is no longer the sole author of “what happens next”.

The continuation is now determined jointly by:

* the current code path,
* the current hardware state,
* the interrupt controller,
* maybe a scheduler above that.

### Epilogue: The Hollywood Principle, badly

People like to summarize event-driven design as:

> **The Hollywood Principle**
> “Don’t call us, we’ll call you.”

In frameworks:

* you don’t own `main`
* you **register callbacks**
* the framework / OS decides *when* to call your code

So:

* *“Don’t call us”* = don’t spin your own loop, don’t poll us constantly
* *“we’ll call you”* = we’ll invoke your handlers when events occur

If we try to map that onto a microcontroller with interrupts:

* “us” in *“don’t call us”* ≈ the **conditions / registers** you might otherwise poll
* “we” in *“we call you”* ≈ the **NVIC / interrupt controller / scheduler** that routes the IRQ to your handler

It’s a bad analogy because:

* it pretends “us” and “we” are the same agent
* it pretends there’s a single well-defined “you”

In reality:

* “you” as in “don’t call us” = your **main loop** code
* “you” as in “we’ll call you” = specific **handlers** bound in some table

Different subjects, different relationships, all flattened into a cute sentence.

From a Lacanian angle, you could say:

* the **Hollywood Principle** names the point where your code acknowledges a **Big Other**:

  * the OS, the scheduler, the framework, the interrupt controller
* you no longer own the master continuation:

  * you place bits of yourself (handlers, callbacks) in its tables,
  * and wait to be **summoned**.

Interrupts and timers are the low-level version of this:

> Hardware to your code:
> “Don’t poll me. I’ll interrupt you when I want your attention.”

And that’s where Time stops being “floating in the air” and becomes:

* oscillators,
* counters,
* interrupts,
* schedulers,

and ultimately: constraints on **which continuation is allowed to run when**.

From here, it’s a short, slightly painful step into the **Scheduler** chapter.
