---
title: Functions
tags: [functions, play]

---

Function arguments, value method:

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


Play method + alternative argument syntax:

~~~
x = { |freq = 440| SinOsc.ar([freq, freq+2], 0, 0.3) }.play;  // the play method uses the function to create a synth on the server, and returns it
x.set(\freq, 480);
x.release(4);                                                 // fadeout over 4 seconds (see documentation)
~~~

Plot and scope methods:

~~~
{ SinOsc.ar(440, 0, 0.2) + Saw.ar(660, 0.2) }.plot(0.01);  // plot 0.01 seconds

{ SinOsc.ar(440, 0, 0.2) + Saw.ar(660, 0.2) }.scope(2);    // plays the function + opens scope window with 2 channels

~~~
