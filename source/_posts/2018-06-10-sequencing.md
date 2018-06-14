---
title: Sequencing
tags: [sequencing, pattern, clock, routine, task, pbind]

---


#### Clocks

Clocks keep track of time and allow tasks to be scheduled for some time in the future.

- `SystemClock`: Runs in seconds.
- `AppClock`: Runs in seconds but has a lower system priority (so it is better for graphic updates and other activities that are not time critical).
- `TempoClock`: Runs in beats per second. Its tempo can be changed, and it is also aware of meter changes.

Musical sequencing will usually use `TempoClock`. There can be many `TempoClock`s all running at different speeds.

One TempoClock is the default, accessed by TempoClock.default:

~~~
t = TempoClock.default;
~~~

#### Scheduling

Print "hello" hello, 5 seconds from now:

~~~
SystemClock.sched(5, { "hello".postln });                        // relative scheduling

SystemClock.schedAbs(SystemClock.beats + 5, { "hello".postln } ); // absolute scheduling
~~~

Inside the scheduling function we access the clock running the function with `thisThread.clock`,
and the current time with `thisThread.clock.beats`.

We can schedule:

- `Function`: if it returns a number, the clock will treat that number as the amount of time before running the function again (use `nil` at the end to prevent this).
- `Routine`: numbers yielded by the routine are treated as the amount of time until resuming the routine `yield` and `wait` are synonims.
- `Task`: tasks are like routines, but they can be paused and resumed (in contrast, once you stop a routine, you can only start it over again from the beginning).


We can `play` routines and tasks, which means we schedule them on a clock:

~~~
(
r = Routine({
    var delta;
    loop {
        delta = rrand(1, 3) * 0.5;
        "Will wait ".post; delta.postln;
        delta.yield;
    }
});
)

TempoClock.default.sched(0, r);

r.play(TempoClock.default);        // equivalent
r.play;                            // equivalent

r.stop;
~~~

The `quant` parameter in `play` lets us specify the exact starting time:

- `play(quant: 4)` - start on the next 4-beat boundary.
- `play(quant: [4, 0.5])` - start on the next 4-beat boundary + a half-beat.

~~~
(
SynthDef(\singrain, { |freq = 440|
    var sig;
    sig = SinOsc.ar(freq, 0, 0.2) * EnvGen.kr(Env.perc(0.01, 2), doneAction: 2);
    Out.ar(0, sig!2);
}).add;

t = Task({
    loop {
        [60, 62, 64, 66, 68].do({ |midi|
            Synth(\singrain, [freq: midi.midicps]);
            0.125.wait;
        });
    }
});
)

t.play(quant: 4);
t.stop;
t.play;    // should pick up with the next note

~~~


#### Sequencing with patterns and `Pbind`

In this example we use patterns to define notes and durations, and then use a Task to play them:

~~~
(
SynthDef(\smooth, {
    arg freq = 440, sustain = 1;
    var sig, env;
    env = EnvGen.kr(Env.linen(0.05, sustain - 0.2, 0.1), doneAction: 2);
    sig = SinOsc.ar(freq, 0, 0.5) * env;
    Out.ar(0, sig!2)
}).add;
)

(
var midi, dur;
midi = Pseq([60, 72, 71, 67, 69, 71, 72, 60, 69, 67], 1).asStream;
dur = Pseq([2, 2, 1, 0.5, 0.5, 1, 1, 2, 2, 3], 1).asStream;
Task({
    var delta;
    while {
        delta = dur.next;
        delta.notNil
    } {
        Synth(\smooth, [freq: midi.next.midicps, sustain: delta]);
        delta.yield;
    }
}).play(quant: 1);
)
~~~


We can simplify the code using `Pbind`:

~~~
(
Pbind(
    \instrument, \smooth,
    \midinote, Pseq([60, 72, 71, 67, 69, 71, 72, 60, 69, 67], 1),
    \dur, Pseq([2, 2, 1, 0.5, 0.5, 1, 1, 2, 2, 3], 1)
).play(quant: 1);
)
~~~

The `Pbind` pattern generates a stream of `Event` objects (see below) using the name-value pairs we provide.

When calling `next` on a `Pbind` stream we must provide an `Event` object - `()` or `Event.new` - that will be filled with the name-value pairs.

~~~
(
p = Pbind(
    \degree, Pseq(#[0,   4,   5,   4], 1),
    \dur,    Pseq(#[0.5, 0.5, 0.5, 1], 1),
).asStream;
)
p.next(());    // ( 'degree': 0, 'dur': 0.5 )
p.next(());    // ( 'degree': 4, 'dur': 0.5 )
~~~


#### Events


`Event` is a subclass of `Dictionary` (it consists of name-value pairs).

~~~
e = (freq: 440, dur: 0.5);

e.at(\freq);          // 440
e[\freq];             // 440
e.freq                // 440

e.put(\freq, 880);    // ( 'freq': 880, 'dur': 0.5 )
e[\freq] = 660;       // ( 'freq': 660, 'dur': 0.5 )
e.freq = 220;         // ( 'freq': 220, 'dur': 0.5 )

e.put(\amp, 0.6);     // ( 'freq': 220, 'dur': 0.5, 'amp': 0.6 )
e.put(\dur, nil);     // ( 'freq': 220, 'amp': 0.6 )
~~~


Events respond to a `play` method:

~~~
( 'degree': 0, 'dur': 0.5 ).play;
~~~

An `Event` specifies an action to be taken in response to `play` and a time increment to be returned in response to `delta`.

There is an event prototype, defined in `Event.default`, which provides default values. Its type is 'note', and its `play` function plays a synth on the server.

When a pattern is played, an `EventStreamPlayer` is created, which reads out the events one by one from the pattern's stream (using a given event prototype as the base), and calls `play` on each.

The 'delta' value in the event determines how many beats to wait until the next event (we typically don't provide it directly, but through 'dur').

Play continues until the pattern stops producing events, or you call `.stop` on the `EventStreamPlayer`.


Some of the Event prototype keys (see <http://doc.sccode.org/Classes/Pbind.html>):

- `\instrument: \default` - the `SynthDef` to be played
- `\sustain`- duration of the synth (time between the gate goes from 1 to 0)
- `\amp` - amplitude of the synth
- `\dur` - time until next event
- `\freq` - base pitch of the synth
- `\detune` - pitch will be freq + detune
- `\pan` - panning position


Instead of *freq* we can provide the following related names:

~~~
freq = (midinote + ctranspose).midicps * harmonic;
~~~


Example:

~~~
(
SynthDef.new(\sine, {
    arg freq=440, amp=0.4, rel=0.3, pan=0;
    var sig, env;
    sig = SinOsc.ar(freq);
    env = EnvGen.kr(Env.perc(0.001, rel), doneAction:2);
    sig = sig * env * amp;
    sig = Pan2.ar(sig, pan);
    Out.ar(0, sig);
}).add;
)


(
p = Pbind(
    \instrument, \sine,
    \dur, Pwhite(0.1, 3, inf),
    \freq,
        Pfuncn({ Array.fill(30, exprand(100, 10000))}, inf).round(50)
        *
        Pfuncn({ Array.fill(30, { rrand(-0.2, 0.2).midiratio })}, inf),
    \amp, 0.01,
    \rel, Pexprand(4, 6, inf),
).play;
)

p.stop;
p.resume;
p.stop;


// Another example

(
p = Pbind(
    \instrument, \sine,
    \dur, Pexprand(0.01, 0.1, inf),
    \freq, Pexprand(50, 3000, inf).round(50) * Pwhite(-0.2, 0.2).midiratio,
    \amp, 0.02,
    \rel, Pexprand(4, 6, inf),
    \pan, Pwhite(-1.0, 1.0, inf),
).play;
)


// - Using \midinote, \harmonic and \detune
// - Using Pif and Pkey for conditional patterns, based on the value of another key
// - Using Pdefn to change patterns on the fly
(
p = Pbind(
    \instrument, \sine,
    \dur, Pexprand(0.01, 0.1, inf),
    \midinote, Pdefn(\mn, 43),
    \harmonic, Pexprand(1, 30 ,inf).round,
    \detune, Pwhite(-4.0, 4.0, inf),
    \amp, Pexprand(0.01, 0.06, inf),
    \rel, Pexprand(4, 6, inf),
    \pan, Pif(Pkey(\harmonic).mod(2) > 0, -1, 1).trace,
).play;
)

Pdefn(\mn, 39);
p.stop;

~~~