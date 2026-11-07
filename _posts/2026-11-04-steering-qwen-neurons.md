---
layout: post
title: "How to do brain surgery on an LLM (sort of)"
date: 2026-11-04
description: Neuron Steering, what is it?
---

Hiya :)

My latest personal project is, essentially, doing brain surgery on an LLM.

I know it sounds a bit drastic when I put it that way, but it's (mostly) true. It's closer to poking a brain with an electrode than cutting anything out, let me explain.

To put it simply, an LLM at its very core is just a bunch of fancy matrix maths wrapped up with probabilities, serving you its best guess of what comes next (one little chunk of text, called a **token**, at a time).

The tiny calculator units that do the actual work are called **neurons**. Each one takes in some numbers, multiplies them by an "importance" weight, adds them all up, and passes the result on to the next layer. The numbers that decide how important each input is (the weights and biases) are called **parameters**, and they're fixed once the model has been trained. A model like the one I'm using, Qwen3.5-4B, has around 4 billion (4B) parameters, spread across hundreds of thousands of neurons bundled into layers. This is an oversimplification, but I think it's good as background info.

**Steering** is where it gets fun. Every neuron has a "direction" it nudges the model's internal state in whenever it fires. Steering means artificially adding that nudge, at a strength I can choose, while the model is generating text. Nothing in the model is permanently changed, and if I turn the nudge off, the model goes right back to normal. Push the right neuron hard enough and the model starts thinking it's a lawyer, or an ant.

I've been experimenting with this just to see what happens, with an end goal of finding neurons I can use to steer how the model behaves (e.g. good vs evil, or a specific personality).

There's some great research out there by [@_lyraaaa_](https://x.com/_lyraaaa_), [@treetowntree](https://x.com/treetowntree), and [@Sauers_](https://x.com/Sauers_) on Twitter/X that goes into more depth on how this works, and their results are very amusing! I really recommend you go check them out :) I've been using Lyra's Neuron Explorer repo to steer the neurons of Qwen3.5-4B myself, all running locally on my own GPU: [lyramakesmusic/neuron-explorer](https://github.com/lyramakesmusic/neuron-explorer)
  
That's all for now but don't worry, I'll be back with more soon (results!) :)