---
title: Functions
tags: [functions, play]

---

Function arguments and the `value` method:

~~~
(
f = {
	arg a, b;
    a - b;
};
)

f.value(5, 3);       // 2
f.value(b:3, a:5);   // 2
~~~

Alternative argument syntax:
~~~
f = { |a, b|  a - b };
~~~


Functions are used in control structures such as `if`, `for`, `while`:

~~~
if (x.isNil, { "x is Nil".postln }, { "x is not Nil".postln });
~~~

~~~
(
	x = [0, 1, 2].choose;
	switch (x,
		0, { "hello".postln    },
		1, { "goodbye".postln  },
		   { "whatever".postln }     // default function
	)
)
~~~

~~~
for (3, 7, { arg i; i.postln });
~~~


`while` takes a test function and a body function that will be evaluated as long as the test function is true:
~~~
i = 0;
while ({ i < 5 }, { i = i+1; i.postln });
~~~

---------------------------------------


#### Special methods

The `play` method uses the function to create a synth on the server,
and returns it (see also [`SynthDef` and `Synth`](/post/synthdef)):

~~~
x = { |freq = 440| SinOsc.ar([freq, freq+2], 0, 0.3) }.play;
x.set(\freq, 480);
x.release(4);        // fadeout over 4 seconds (see documentation)
~~~

The `plot` method takes a length in seconds and plots the waveform
defined by the function:

~~~
{ SinOsc.ar(440, 0, 0.2) + Saw.ar(660, 0.2) }.plot(0.01);
~~~

The `scope` method plays the function and opens the scope window with
the specified number of channels:
~~~
{ SinOsc.ar(440, 0, 0.2) + Saw.ar(660, 0.2) }.scope(2);
~~~