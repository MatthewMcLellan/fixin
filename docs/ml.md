# The ML behind the diagnosis

## The problem

People photograph what's broken: a leaking water heater, a warning light, a corroded battery terminal on a boat. fixin sorts each photo into a tree:

```
domain    →  car · home · boat
category  →  e.g. Brakes, HVAC, Electrical, Fuel
fault     →  e.g. Worn brake pads, Register blocked, Tank vent blocked
```

The tree has more than 2,300 classes at the bottom. Getting the domain and category right matters most, because that's what tells you how worried to be and who to call.

## Data

The training corpus is about **500,000 images (146 GB)** across **2,300+ classes**. An unattended pipeline builds it:

1. It finds categories where the model is weakest and searches for new images of them.
2. It removes near-duplicates by perceptual hash, so the same photo can't land in both the training and the test set.
3. It scores each image against its label with **CLIP** and drops the ones that don't match.
4. It keeps a provenance record for every image.

## From 118 models to one network

The first engines were a **cascade**: one classifier picked the domain, another the category, and a separate small model for each category picked the fault. That added up to 118 models. Ruby 3.9 is the cascade engine still offered.

The Sapphire generation replaced the cascade with a single **MobileViTv2** network with **117 output heads**, one for each decision in the tree, all sharing one backbone.

- It beat the cascade at every level of the tree.
- It cut the per-phone download from about **330 MB to about 42 MB**.
- On Android, engines ship as float16 TensorFlow Lite at about 12 MB each.

Training runs in **PyTorch** on NVIDIA GPUs. Each engine is then converted to **Core ML** and **TensorFlow Lite** and checked against the original before it ships.

## Confidence you can act on

A raw softmax score is a poor guide, because a network can sound sure and still be wrong. So:

- **Per-head temperature scaling.** Each of the heads gets its own temperature, fitted on a held-out split, so its confidence matches how often it's actually right.
- **Words instead of percentages.** Results say *Strong match*, *Possible match — worth a second look* or *Low match — retake in better light?*
- **Alternatives when it's close.** When the leader is weak, or the runner-up is within a calibrated margin, the result lists the close candidates instead of forcing one answer. Top-three accuracy runs about 20 points above top-one, so the shortlist usually contains the right answer.

## Measuring honestly

Every engine is scored on a **frozen test set** that never changes, so generations can be compared fairly.

- **Naming the exact fault is much harder than naming the category.** The app calls the fault a *specific guess*, and the iPhone App Clip stops at the category on purpose.
- **Newer doesn't mean better.** Sapphire 4.4 scored below 4.3 on the frozen set, so it was held back and later withdrawn.

## Engine generations

| Engine | How it works | Plans |
| --- | --- | --- |
| Ruby 3.9 | Cascade of smaller classifiers | Every plan |
| Sapphire 4.3 | One MobileViTv2 network with many heads | Every plan (free default) |
| Sapphire 4.8 | Better at telling similar problems apart | Premiere, The One (paid default) |
| Emerald 5.0 / 5.2 | Newest generation, 13 more kinds of problem | Premiere, The One |
