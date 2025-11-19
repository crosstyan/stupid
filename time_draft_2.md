Time

So far we’ve been pretending the machine just… runs.

No one asked:
“Runs at what speed?”
“Against what clock?”
“Who says ‘now’?”

Everything we’ve seen so far floats in a timeless symbolic space:
types, pointers, stacks, even assembly.

welcome to the desert of the real

Clock (cycles are not seconds)

A CPU doesn’t “know” what a second is.

What it has is:
	•	a clock source (crystal oscillator / RC / PLL)
	•	a frequency (“168 MHz”, “48 MHz”, etc.)

Internally it just hears:

tick, tick, tick, tick…

Each tick says:
“you may advance your internal state once.”

Humans put a story on top of it:
	•	“this loop runs every 1 ms”
	•	“this function takes 5 µs”
	•	“this task has a 10 ms deadline”

For the hardware, it’s just “I’ve seen N ticks since reset”.

The Real here is the physical oscillator:
jitter, drift, temperature, supply noise.
Your clean mental picture (“1 cycle per instruction”) is already a fantasy.

Time base (counting something)

To say “sleep 10 ms” you need a time base:
	•	some counter incrementing at a known rate
	•	some way to compare it with a target

The shape is always:

[clock] → [divider] → [counter] → [compare]

This can be:
	•	a 555 timer
	•	a hardware timer peripheral (SysTick, TIM2…)
	•	TSC / HPET / APIC timer on a PC

The CPU itself doesn’t feel “10 ms passed”.
It only sees: “some counter reached some value”.

Time, for software, is always mediated by some counting device.

Interruption (first crack in von Neumann)

Up to now, our control flow was:
	•	fetch instruction at PC
	•	execute
	•	increment PC
	•	repeat

This is the pure von Neumann loop:
one linear story, driven entirely by your code.

Introduce a timer:
	1.	configure it to fire every N ticks
	2.	when its counter hits N, it raises a signal
	3.	the CPU jumps to an interrupt handler

Now something new happens:
	•	the PC changes to a new address
	•	not because your program asked for it (call, goto)
	•	but because the outside asserted an interrupt line

Your neat Symbolic order (“this line, then that line”) is cut by an event it didn’t schedule.

In Lacanian terms: the Real intrudes:
	•	your code has no internal reason “why now”
	•	the timer just says: time’s up, and you obey

Microcontroller version: SysTick_Handler suddenly runs.
You didn’t call it; it called you.

Timer (the simplest peripheral)

A timer is probably the tiniest “other agent” on the chip:
	•	you write its config registers
	•	it counts clock pulses
	•	maybe divides them
	•	compares against a value
	•	optionally raises an interrupt

From 555 to SysTick, it’s all variations of:

clock → counter → compare → (maybe) IRQ

Why is this necessary?

Because:
	•	CPU cycles are local (“I ran N instructions”)
	•	real-world time is global (“10 ms have passed, motors moved, inputs changed”)

If you want:
	•	periodic sampling
	•	animations
	•	timeouts
	•	pre-emptive multitasking

you need something that says: “regardless of what your code is doing, now is now”.

The CPU doesn’t know time;
it only knows “an interrupt occurred”.

Push / Pull (and their duality)

There are two basic ways to deal with the world:

Pull:

“I’ll ask when I feel like it.”

	•	polling a GPIO in a loop
	•	checking millis() and seeing if enough time passed
	•	if (uart_rx_ready()) read();

Push:

“The world tells me when it has something to say.”

	•	timer interrupt every 1 ms
	•	UART RX interrupt on byte received
	•	external interrupt on rising edge

They are dual:
	•	you can turn push into pull:
ISR just sets a flag; main loop polls the flag.
	•	you can turn pull into push:
main loop invokes callbacks when something changes.

But interrupts + timers force you to acknowledge:

you are not the only author of “what happens next”.

The continuation of your program is now jointly chosen by:
	•	your own control flow,
	•	the timer / peripherals,
	•	and whoever owns the interrupt controller.

Epilogue: “Don’t call us, we’ll call you”

There’s a well-known object-oriented slogan:

The Hollywood Principle:
“Don’t call us, we’ll call you.”

In GUI frameworks / event loops / OS APIs, that means:
	•	you don’t control the main loop,
	•	you register callbacks / handlers,
	•	the framework / OS decides when to invoke them.

So, who is “we” here?

In an MCU:
	•	“we” ≈ NVIC + vector table + your ISR wiring
	•	“us” ≈ the code you think you’re writing linearly

In an OS:
	•	“we” ≈ scheduler + kernel + event loop
	•	“us” ≈ your process, your handlers, your threads

It’s a bad analogy if you take it literally:
	•	the conditions (timer reaching N, GPIO line toggling)
are not the same “person” as
	•	the entity dispatching handlers (NVIC, kernel, framework)

There is no symmetric phone call here.
Your code doesn’t really “call them”, and they don’t politely “call you back”.
It’s closer to:

the Big Other reserves the right to interrupt you whenever it wants.

The “we” is the Other that owns the main loop.
You’re just a handler it may or may not run.

In Lacanian flavour:
	•	your program is the speaking subject, telling its little imperative story
	•	the timer/interrupt/scheduler ensemble plays the role of the Big Other:
	•	it decides when your turn to speak is over,
	•	it decides when another handler gets CPU time,
	•	it decides when your continuation is suspended or resumed.

You don’t get to say “I’ll run forever”.
At some point, the hardware, the OS, or the runtime says:

“Don’t loop us, we’ll interrupt you.”

And that’s the moment Time really enters your model.
