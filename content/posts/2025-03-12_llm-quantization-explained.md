+++
title = "LLM Quantization, Explained in Plain English"

[taxonomies]
tags = ["AI", "Machine Learning"]
+++

If you've tried running a large language model on your own hardware, you've
probably run into the same wall everyone does: the model is huge, your GPU
memory is not, and downloading a "7B" or "70B" model suddenly feels like a
math problem. Quantization is the main trick the community uses to close
that gap.

<!-- more -->

## What quantization actually does

A neural network is just a giant pile of numbers — weights — that get
multiplied and added together. By default, those numbers are usually stored
as 32-bit or 16-bit floating point values (FP32 / FP16 / BF16). Quantization
takes those weights and represents them with fewer bits, most commonly
8-bit or 4-bit integers.

Fewer bits per weight means:

- **Less memory** — a 7-billion parameter model at FP16 needs roughly 14GB
  just to hold the weights. Quantized to 4-bit, that drops to around 4GB.
- **Faster loading and often faster inference** — less data to move around
  means less time spent waiting on memory bandwidth, which is usually the
  actual bottleneck, not raw compute.
- **Some loss in precision** — you're approximating each weight with a
  coarser number, so the model's output can drift slightly from the
  original.

The whole game in quantization research is minimizing that last point while
maximizing the first two.

## Not all quantization is the same

A few approaches you'll run into:

- **Post-training quantization (PTQ)** — take an already-trained model and
  quantize its weights afterward, without retraining. This is the fastest
  and most common approach for running models locally.
- **Quantization-aware training (QAT)** — the model is trained (or
  fine-tuned) while simulating low-precision arithmetic, so it learns to
  compensate for the rounding error. More expensive, but usually more
  accurate at very low bit-widths.
- **Weight-only vs. weight-and-activation quantization** — some methods
  only shrink the stored weights and compute in higher precision at
  runtime; others also quantize the activations (the intermediate values
  flowing through the network), which saves more memory and compute but is
  harder to get right.

Popular formats you'll see in the wild — GGUF (used by llama.cpp), GPTQ,
and AWQ — are all just different recipes for doing PTQ well, each making
different trade-offs about which weights get how much precision and how
outliers are handled.

## How low can you go?

8-bit and 4-bit quantization are well-established and usually cause only a
small, often unnoticeable drop in output quality. Researchers have also
pushed into more extreme territory — 2-bit, and even ternary
representations where each weight is restricted to just three possible
values (-1, 0, or 1). At that extreme, you need architectural changes or
training the model with quantization in mind from the start, because
simply rounding an existing model's weights to three values throws away too
much information to be useful.

## Why this matters if you're just running models locally

You don't need to understand the math behind GPTQ to benefit from it. In
practice, the workflow looks like:

1. Pick a model.
2. Pick a quantization level based on your hardware (4-bit is a common
   sweet spot for consumer GPUs).
3. Download the pre-quantized weights — someone has usually already done
   the quantization for you — and run it with a tool like llama.cpp,
   Ollama, or the `transformers` + `bitsandbytes` stack.

The practical rule of thumb: if a model "doesn't fit," before giving up or
buying a bigger GPU, check whether a quantized version exists. Nine times
out of ten, it does.
