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

Sustaining envelopes have a sustain level (output stays there until a trigger signal is received, then it transitions to the release node).

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

~~~
{ PinkNoise.ar(EnvGen.kr(Env.perc, doneAction: 2)) }.play
~~~
