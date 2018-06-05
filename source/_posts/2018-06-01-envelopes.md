---
title: Envelopes and Triggers
tags: [ugen, env, trigger, play]

---

#### Envelopes


An `Env` is a specification for an envelope (a shape constructed by segments).

It is created by specifying:

- an array of levels (n)
- an array of duration for each segment (n - 1)
- an array of curvatures for each segment (n -1)

Curvatures can be defined by:

- floats:
	- = 0: linear
	- > 0: curve up
	- < 0: curve down
- symbols:
	- `\lin`
	- `\exp`
	- `\sqr`
	- ...


~~~
Env.new([0, 1, 0.5, 0], [1, 2, 5], [4, 2, -4]).plot;
~~~

![env](/images/blog/env.png "Env")


There are also some shortcut methods for creating envs, such as:

`Env.perc`

- attack time
- release time
- peak level
- curvature

~~~
Env.perc(0.02, 1.5, 1, -4).plot;
~~~

![perc](/images/blog/perc.png "Env.perc")


### Sustaining envelopes

Sustaining envelopes have a sustain level (output stays there until a trigger signal is received, then transitions to the release node).

`Env.adsr`

- attackTime: 0.01
- decayTime: 0.3
- sustainLevel: 0.5
- releaseTime: 1

![adsr](/images/blog/adsr.png "Env.adsr")

`Env.asr`

- attackTime: 0.01
- sustainLevel: 1
- releaseTime: 1

![asr](/images/blog/asr.png "Env.asr")


------------------------------------------

#### EnvGen

In the server we use envelopes created by `Env` through the `EnvGen` UGen.

It takes an `Env` as the first parameters, and then we can also provide:

- gate: 0 (used to trigger sustaining envelopes - see *triggers* below)
- doneAction: 0

The doneAction argument is used to specify what to do when the env is finished playing. Mainly we'll use these two values:

- 0: do nothing
- 2: free the enclosing synth


We can use an `EnvGen` in place of an amplitude:

~~~
{ PinkNoise.ar(EnvGen.kr(Env.perc, doneAction: 2)) }.play
~~~

Or we can multiply a UGen by an `EnvGen`:

~~~
{ PinkNoise.ar() * EnvGen.kr(Env.perc, doneAction: 2) }.play
~~~

------------------------------------------

#### Triggers

In general, triggers are signals that, when change from a non-positive value to a positive value or vice versa, can affect UGens in some way.

As we've seen, `EnvGen` accepts a `gate` variable. We can control the state of a sustaining envelope by assign a trigger signal to that `gate` variable.

By default `gate` is 1, which means the envelope will start playing, and will keep playing at the sustain level.

When `gate` reaches 0, the envelope is released, which means it continues from its sustain level down to the end of the envelope.

To retrigger the envelope, we must provide a positive value for the gate, and so on.

~~~
(
SynthDef(\sawdsr, {
    arg gate = 0, freq = 440, doneAction = 0;
    var sig =  LFTri.ar(freq, 0.4);
    sig = sig * EnvGen.kr(Env.adsr(0.03, 0.2, 0.5, 2), gate: gate, doneAction: doneAction);
    Out.ar(0, sig!2);
}).add;
)

a = Synth(\sawdsr);               // create the synth (does not play)

a.set(\gate, 1, \freq, 300);      // play
a.set(\gate, 0);                  // release

a.set(\gate, 1, \freq, 320);      // retrigger
a.set(\gate, 0);                  // release

a.set(\gate, 1, \freq, 340);      // retrigger
a.release(3);                     // release (convenience method, overrides duration)

a.set(\gate, 1, \freq, 360);      // retrigger
a.set(\gate, 0, \doneAction, 2);  // release and free the synth
~~~


Example with `Routine` and `wait`:

~~~
(
SynthDef.new(\spooky, {
	arg amp = 0.2, gate = 1;
	var sig = 0, env, temp;
	env = EnvGen.kr(
        Env.adsr(0.5, 0.5, 0.7, 10),
		gate,
		doneAction: 2
	);
	10.do({
		arg i;
		var freq_variation = LFNoise1.kr(0.5).exprange(0.99, 1.01); // slow detune
        temp = SinOsc.ar({ ExpRand.new(200,2000) }*freq_variation!2, 0, amp); // a random sinewave
		temp = temp * LFNoise1.kr(0.4).exprange(0.05, 1);  // slow amplitude modulation
		sig = sig + temp;
	});
	sig = sig * env;
	Out.ar(0, sig);
}).add;
)

(
Routine.new({
    10.do({
        var synth = Synth.new(\spooky, [\amp, 0.05]);
        10.wait;
        synth.set(\gate, 0);
        5.wait;
    });
}).play;
)
~~~

Using another UGen as a gate trigger: in this example, `Dust`, which generates random impules (0 to 1) at a given average number of impulses per second (density):

~~~
(
SynthDef(\shots, {
    arg density = 0.5;
    var sig =  PinkNoise.ar(0.5);
    sig = sig * EnvGen.kr(Env.perc, gate: Dust.kr(density));
    Out.ar(0, sig!2);
}).add;
)

a = Synth(\shots, [\density, 2]);
~~~


Note: in a `SynthDef` function, arguments that begin with *t_* will be made as `TrigControl`s, which means that setting the argument to 1 will create a control-rate impulse, that will automatically go back to zero.
(See <http://doc.sccode.org/Classes/SynthDef.html>)