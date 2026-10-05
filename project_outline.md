# Image Segmentation

## 1. What is the competition actually asking?

Imagine you have a factory producing metal/industrial parts.

A camera takes a picture of each product:

```text
        Industrial Image
              ↓
        ┌─────────────┐
        │             │
        │    ███      │  ← defect
        │             │
        │        ██   │  ← another defect
        └─────────────┘
```

Your AI has to answer **two questions simultaneously**:

### Question 1 — What type of defect is present?

There are **4 defect categories**:

```text
defect_a
defect_b
defect_c
defect_d
```

### Question 2 — Exactly which pixels contain that defect?

This is the important part.

You are **not** doing only:

```text
Image → "defect_a"
```

You need:

```text
Image
  ↓
Which defect?
  ↓
Where exactly are its pixels?
```

So this is a **semantic segmentation problem with 4 defect categories**.

---

# 2. Classification vs Segmentation

This distinction is extremely important.

### Classification

The model might say:

```text
Image → defect_a
```

That's all.

You don't know where the defect is.

### Segmentation

The model says:

```text
Image
 ↓
┌──────────────────┐
│                  │
│     █████        │
│     █████        │
│                  │
└──────────────────┘
```

It gives you a **pixel-level mask**.

So this competition is essentially:

> **Detect the defect AND draw a mask around every pixel belonging to that defect.**

---

# 3. What does the training data look like?

You have something like:

| image_id | defect_type | mask_rle              |
| -------- | ----------- | --------------------- |
| IMG001   | defect_a    | `3201 12 3329 12 ...` |
| IMG001   | defect_b    | empty                 |
| IMG001   | defect_c    | empty                 |
| IMG001   | defect_d    | `5000 20 ...`         |
| IMG002   | defect_a    | empty                 |
| IMG002   | defect_b    | `1200 15 ...`         |
| ...      | ...         | ...                   |

Notice something important:

For **every image**, you have **4 rows**.

Because there are 4 defect types.

So conceptually:

```text
IMG001
 ├── defect_a → mask
 ├── defect_b → no defect
 ├── defect_c → no defect
 └── defect_d → mask
```

---

# 4. Why are there 4 rows per image?

Because the competition wants you to independently answer:

> "Does this particular image contain defect A, and if yes, where?"

Then:

> "Does it contain defect B, and if yes, where?"

And so on.

Therefore:

```text
image × defect_type
```

is the fundamental prediction unit.

For one image:

```text
              Image
                │
       ┌────────┼────────┐
       ↓        ↓        ↓
   defect_a  defect_b  defect_c  defect_d
       ↓        ↓        ↓        ↓
      mask     mask     mask     mask
```

---

# 5. What does "multiple disconnected regions" mean?

Suppose `defect_a` appears in three different places:

```text
┌───────────────────────┐
│                       │
│   ███                 │
│   ███                 │
│                       │
│             █████     │
│             █████     │
│                       │
│       ██              │
│       ██              │
└───────────────────────┘
```

All three regions belong to **the same defect category**.

You still need **one mask** for `defect_a`.

So your model should produce something like:

```text
defect_a → [region 1 + region 2 + region 3]
```

It doesn't have to be one connected blob.

---

# 6. What exactly does the model output?

During prediction, imagine your model receives:

```text
image
```

and produces **4 segmentation masks**:

```text
                 Model
                   ↓
        ┌──────────┼──────────┐
        ↓          ↓          ↓
    Mask A      Mask B      Mask C      Mask D
```

For example:

```text
Mask A:
0 0 0 0 0
0 1 1 0 0
0 1 1 0 0
0 0 0 0 0

Mask B:
0 0 0 0 0
0 0 0 0 0
0 0 0 0 0
0 0 0 0 0
```

Here:

```text
Mask A → defect exists
Mask B → no defect
```

And similarly for C and D.

---

# 7. Why does the competition use RLE?

A mask could potentially contain millions of pixels.

Instead of submitting:

```text
0 0 0 0 0 0 0 1 1 1 1 1 0 0 ...
```

they use **Run Length Encoding (RLE)**.

Suppose a flattened mask looks like:

```text
0 0 0 1 1 1 1 1 0 0
```

Instead of writing all pixels, we say:

```text
4 5
```

Meaning:

```text
start = 4
length = 5
```

So pixels:

```text
4,5,6,7,8
```

are foreground.

---

# 8. Multiple regions in RLE

Suppose:

```text
0 0 0 1 1 1 0 0 1 1 0
```

We have:

```text
region 1 → start 4, length 3
region 2 → start 9, length 2
```

So:

```text
4 3 9 2
```

That's why you can see something like:

```text
3201 12 3329 12 3457 12
```

It means:

```text
start  length
3201   12
3329   12
3457   12
```

---

# 9. What does "column major ordering" mean?

This is **VERY important** for RLE.

Suppose your image is:

```text
A B C
D E F
G H I
```

Normally, many programming operations think row-wise:

```text
A B C D E F G H I
```

But **column-major** means reading columns first:

```text
A D G B E H C F I
```

So the RLE positions are based on:

```text
A
D
G
B
E
H
C
F
I
```

not:

```text
A
B
C
D
E
F
G
H
I
```

This matters when converting:

```text
model mask → RLE
```

and also when converting:

```text
RLE → mask
```

If you get this wrong, your submission mask will be completely misplaced.

---

# 10. Now let's understand the evaluation

This is probably the most important part of the competition.

Your final score is made of **three things**:

```text
             Final Score
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
    Dice 70%    FP 20%    Empty 10%
```

---

# 11. Component 1 — Foreground Dice = 70%

This is the biggest component.

Suppose the actual defect is:

```text
Ground Truth

██████
██████
```

Your prediction:

```text
Prediction

 █████
 █████
```

There is a large overlap.

So Dice is high.

The formula is:

$$
Dice = \frac{2|X \cap Y|}{|X|+|Y|}
$$

where:

```text
X = predicted defect pixels
Y = actual defect pixels
```

### Perfect prediction

```text
Prediction = Ground Truth
```

Dice:

```text
1.0
```

### No overlap

```text
Prediction:  ███

Ground truth:       ███
```

Dice:

```text
0
```

Therefore:

> **You want your predicted defect mask to overlap the real defect mask as accurately as possible.**

And because Dice contributes **70%**, this is your biggest priority.

---

# 12. Component 2 — False Positive Score = 20%

Now suppose there is **no defect_a**.

Ground truth:

```text
defect_a → empty
```

But your model predicts:

```text
defect_a → █████
```

That's a **false positive**.

You're basically saying:

> "There is a defect here!"

when there isn't one.

The competition penalizes this.

So your model shouldn't simply predict defects everywhere.

---

# 13. Component 3 — Empty Prediction Accuracy = 10%

This is the opposite situation.

Suppose:

```text
Ground truth:

defect_b → no defect
```

and your model correctly predicts:

```text
defect_b → empty
```

That's good.

You get credit for correctly identifying that the defect category isn't present.

---

# 14. Therefore, the model needs BOTH skills

This competition isn't simply:

> "Find as many defects as possible."

Because then your model might predict:

```text
defect_a → defect everywhere
defect_b → defect everywhere
defect_c → defect everywhere
defect_d → defect everywhere
```

That would destroy your false-positive performance.

Instead you need:

### Skill 1

When a defect exists:

```text
Find it accurately.
```

### Skill 2

When a defect doesn't exist:

```text
Predict EMPTY.
```

So conceptually:

```text
                 IMAGE
                   ↓
            ┌──────────────┐
            │     MODEL    │
            └──────────────┘
                   ↓
        ┌──────────┴──────────┐
        ↓                     ↓
   Is defect present?      No defect
        ↓                     ↓
       YES                   EMPTY
        ↓
   Find exact pixels
```

---

# 15. Why small defects are difficult

The statement says defects vary in:

* size
* shape
* position
* orientation
* texture
* contrast
* number of occurrences

Imagine two images.

### Large defect

```text
████████████
████████████
████████████
```

Easy-ish.

### Tiny defect

```text
................
................
.......██.......
................
................
```

Much harder.

If your image is `512 × 512`, that tiny defect might occupy only a few hundred pixels.

So your model needs to preserve **fine spatial information**.

This is one reason segmentation architectures such as:

```text
U-Net
U-Net++
DeepLab
SegFormer
Mask2Former
```

can be useful.

But **don't choose the architecture yet**. First understand the data and metric.

---

# 16. The most important mental model

Think of the competition as **4 binary segmentation problems happening simultaneously**.

Instead of thinking:

```text
Image → 4-class classification
```

think:

```text
Image
 │
 ├── Is defect A present?
 │       └── If yes → pixels of A
 │
 ├── Is defect B present?
 │       └── If yes → pixels of B
 │
 ├── Is defect C present?
 │       └── If yes → pixels of C
 │
 └── Is defect D present?
         └── If yes → pixels of D
```

So your model effectively needs:

```text
Image
  ↓
4 masks
  ↓
┌────────┬────────┬────────┬────────┐
│ Mask A │ Mask B │ Mask C │ Mask D │
└────────┴────────┴────────┴────────┘
```

---

# 17. What happens during training?

Suppose training image is:

```text
image_001.jpg
```

and its annotations are:

```text
image_001 | defect_a | RLE
image_001 | defect_b | empty
image_001 | defect_c | RLE
image_001 | defect_d | empty
```

First, you decode the RLE into actual masks:

```text
RLE
 ↓
binary mask
```

You now have:

```text
image
+
mask_a
+
mask_b
+
mask_c
+
mask_d
```

Then:

```text
             Image
               ↓
             Model
               ↓
       ┌───────┼───────┐
       ↓       ↓       ↓
    pred_A   pred_B   pred_C   pred_D
       ↓       ↓       ↓       ↓
    compare  compare compare compare
       ↓
      loss
       ↓
   backpropagation
```

The model learns to produce masks similar to the ground-truth masks.

---

# 18. What happens during testing?

For an unseen image:

```text
test_image_001
```

there is no ground truth.

You do:

```text
test image
     ↓
   model
     ↓
4 predicted masks
     ↓
convert masks → RLE
     ↓
submission.csv
```

---

# 19. Why does the submission need 4 rows per image?

Suppose test contains:

```text
IMG001
IMG002
```

You need:

```text
IMG001 × defect_a
IMG001 × defect_b
IMG001 × defect_c
IMG001 × defect_d

IMG002 × defect_a
IMG002 × defect_b
IMG002 × defect_c
IMG002 × defect_d
```

Therefore:

```text
Number of submission rows
=
number of test images × 4
```

assuming exactly four defect categories.

---

# 20. Example submission

Suppose:

```text
IMG001
```

has:

```text
defect_a → detected
defect_b → absent
defect_c → detected
defect_d → absent
```

Then:

```text
ID,image_id,defect_type,mask_rle

1,IMG001,defect_a,"3201 12 3329 12"
2,IMG001,defect_b,""
3,IMG001,defect_c,"5000 20 6000 10"
4,IMG001,defect_d,""
```

The empty strings mean:

```text
"No pixels predicted for this defect."
```

---

# 21. One subtle but VERY important point

Don't confuse:

### Image-level absence

```text
This image has no defects at all.
```

with:

### Category-level absence

```text
This image has defects,
but not defect_b.
```

For example:

```text
Image:
 ├── defect_a → YES
 ├── defect_b → NO
 ├── defect_c → YES
 └── defect_d → NO
```

The image is **not empty**.

But the masks for B and D are empty.

This is exactly why the submission is based on:

```text
image_id × defect_type
```

rather than simply one prediction per image.

---

# 22. So what is our actual ML task?

In one sentence:

> **Given an industrial image, predict a pixel-level binary mask independently for each of four defect categories, and convert those masks into column-major RLE format for submission.**

Or even simpler:

```text
IMAGE
  ↓
       ┌───────────────┐
       │ AI SEGMENTER  │
       └───────────────┘
          ↓ ↓ ↓ ↓
         A B C D
          ↓ ↓ ↓ ↓
       4 binary masks
          ↓
         RLE
          ↓
    submission.csv
```

---

## 23. What I would focus on first

**Don't start with CNN architecture yet.** For this competition, I'd understand the pipeline in this order:

```text
STEP 1
Understand images
        ↓
STEP 2
Understand train.csv
        ↓
STEP 3
Understand RLE
        ↓
STEP 4
RLE → binary mask
        ↓
STEP 5
Visualize image + mask
        ↓
STEP 6
Understand train/validation split
        ↓
STEP 7
Build a simple segmentation baseline
        ↓
STEP 8
Calculate Dice locally
        ↓
STEP 9
Improve model
        ↓
STEP 10
binary mask → RLE
        ↓
STEP 11
Create submission
        ↓
STEP 12
Kaggle submission
```

### The **first practical thing** I would do

Before deciding between U-Net, SegFormer, DeepLab, etc., take **5–10 training images** and make a visualization like:

```text
┌──────────────┐
│ Original     │
│ Image        │
└──────────────┘
       +
┌──────────────┐
│ Ground Truth │
│ Defect Mask  │
└──────────────┘
```

for each of the four defect types.

That will immediately tell us **what kind of anomalies we're dealing with**, how large they are, whether the classes are imbalanced, and whether tiny-defect handling is going to be the main challenge.
