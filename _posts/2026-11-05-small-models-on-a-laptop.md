---
layout: post
title: Running Small AI Models on a Laptop
date: 2026-11-05 08:00:00 +0800
description: A short note on running small models (GPT-2, CLIP) locally on an ordinary laptop with no GPU — what they can do, how many parameters they have, and how much memory they need.
tags: ai tools
categories: blog
---

To understand AI models better, I recommend trying small models that can be installed on a laptop. You do not need a GPU to play with real neural networks. My laptop (Intel i7-1260P, 32 GB RAM, no GPU) runs the two models below in a few seconds each. All it takes is Python with the CPU-only build of PyTorch and Hugging Face `transformers`; the weights are downloaded once and then everything runs offline.

| Model                                 | What it can do                                                        | Parameters                                                    | Disk   | Peak RAM |
| ------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------- | ------ | -------- |
| GPT-2                                 | Writes text by predicting the next word                               | 124M (12 transformer blocks + a 50,257 × 768 token embedding) | 0.5 GB | 0.9 GB   |
| CLIP (`openai/clip-vit-base-patch32`) | Picks the caption that best matches an image, using labels you choose | 151M (87M image encoder + 63M text encoder)                   | 0.6 GB | 0.9 GB   |

Rule of thumb: memory ≈ number of parameters × 4 bytes (float32), plus a few hundred MB for Python and PyTorch, so models up to about 1 billion parameters (4 GB) are comfortable on an ordinary laptop.

## How they run

**GPT-2.** The text is split into tokens, and each token becomes a 768-dimensional vector. These vectors pass through the 12 transformer blocks, where each token mixes in information from the tokens before it. The last vector is turned into a probability for each of the 50,257 tokens. One token is sampled, added to the text, and the process repeats. To try it, run the following Python code:

```python
from transformers import AutoTokenizer, AutoModelForCausalLM

tok = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")

ids = tok("The theory of algebraic curves", return_tensors="pt").input_ids
probs = model(ids).logits[0, -1].softmax(-1)  # probability of each next token
for p, i in zip(*probs.topk(3)):
    print(repr(tok.decode(i)), round(p.item(), 3))  # ' is' 0.326, ',' 0.073, ' and' 0.046
print(tok.decode(model.generate(ids, max_new_tokens=30, do_sample=True)[0]))  # sample, append, repeat
```

**CLIP.** The image is resized to 224 × 224 and cut into 49 patches of 32 × 32 pixels. An image transformer turns the patches into one 512-dimensional vector, and a text transformer does the same for each caption. The caption whose vector is closest to the image's (by cosine similarity) wins. To try it, save the following script as `classify.py`:

```python
import sys
from PIL import Image
from transformers import CLIPModel, CLIPProcessor, logging

if len(sys.argv) < 3:
    sys.exit('usage: python classify.py IMAGE "label 1" "label 2" ...')
image = Image.open(sys.argv[1])
labels = sys.argv[2:]

logging.set_verbosity_error()
name = "openai/clip-vit-base-patch32"
model = CLIPModel.from_pretrained(name)
proc = CLIPProcessor.from_pretrained(name)

inputs = proc(text=labels, images=image, return_tensors="pt", padding=True)
logits = model(**inputs).logits_per_image[0]
cosine = logits / model.logit_scale.exp()  # raw similarity between image vector and text vector
probs = logits.softmax(-1)                 # softmax turns small cosine gaps into big probability gaps

print(" prob  cosine  label")
for p, c, label in sorted(zip(probs.tolist(), cosine.tolist(), labels), reverse=True):
    print(f"{p:6.1%}  {c:.3f}  {label}")
```

and run it in the terminal with an image and some candidate labels:

```console
$ python classify.py images/cats.jpg "a photo of cats" "a photo of a dog" "a commutative diagram"
 prob  cosine  label
 99.9%  0.275  a photo of cats
  0.1%  0.199  a commutative diagram
  0.0%  0.193  a photo of a dog
```

Almost all of this is matrix multiplication, which is why an ordinary CPU can do it in seconds.
