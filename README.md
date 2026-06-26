# TakeMeter

A classifier that sorts music-discussion posts by how strongly the take is supported — from evidence-backed argument, to bold unargued opinion, to pure emotional reaction.


TakeMeter classifies posts from [r/LetsTalkMusic](https://www.reddit.com/r/LetsTalkMusic/) — a subreddit built around in-depth music discussion into one of the 3 following labels:

- **`analysis`** — a structured argument backed by specific, verifiable evidence (musical detail, named examples, history, data) that stands on its own without the opinion framing.
- **`hot_take`** — a bold evaluative claim or generalization that's asserted rather than argued; evidence is absent
- **`reaction`** — a personal or emotional response to a song or artist

I chose this community because its discourse genuinely varies in form: a sample of posts spanned evidence-backed arguments, confident unsupported claims, and emotional reactions.

The 2 boundaries that cause real ambiguity, and the rules I used to resolve them:

- **`analysis` vs. `hot_take`** — a post can sound argued without being argued. Rule: if specific, verifiable evidence would support the claim even after getting rid of the opinion framing → `analysis`; if the support is decorative → `hot_take`.
- **`hot_take` vs. `reaction`** — an emotional post can smuggle in a claim. Rule: a general claim about the music → `hot_take`; only a personal feeling → `reaction`.

When a post is genuinely 50/50, I default to the lower-evidence label (`reaction` > `hot_take` > `analysis`) to keep the `analysis` class clean.

## Data

- **Source:** Reddit's official API via PRAW, pulling from `new`, `hot`, and `top`/`controversial`. (PullPush was a stale fallback only.)
- **Size:** ~200 labeled examples
- **Collection & filtering:** Out-of-scope posts (recommendation requests, pure meta/off-topic threads) were filtered at collection rather than given a junk label. 
- **Annotation:** Each post was pre-labeled with AI assistance, after which I checked them manually based on the rules

## Method

- **Model:** base model - distilbert-base-uncased, fine-tuned on the labeled set.
- **Split:** train/validation/test split --> (70% / 15% / 15%)
- **Baseline LLM:** Groq (llama-3.3-70b-versatile)


## Evaluation

Baseline accuracy: 0.517  (evaluated on 29/30 parseable responses)

Per-class metrics (baseline):
              precision    recall  f1-score   support

    analysis       0.17      0.20      0.18         5
    hot_take       0.40      0.33      0.36         6
    reaction       0.67      0.67      0.67        18

    accuracy                           0.52        29
   macro avg       0.41      0.40      0.40        29
weighted avg       0.53      0.52      0.52        29



Fine-tuned model accuracy: 0.600

Per-class metrics (fine-tuned model):
              precision    recall  f1-score   support

    analysis       0.00      0.00      0.00         5
    hot_take       0.00      0.00      0.00         7
    reaction       0.60      1.00      0.75        18

    accuracy                           0.60        30
   macro avg       0.20      0.33      0.25        30
weighted avg       0.36      0.60      0.45        30

==================================================
RESULTS COMPARISON
==================================================
Model                               Accuracy
---------------------------------------------
Zero-shot baseline (Groq)              0.517
Fine-tuned DistilBERT                  0.600
---------------------------------------------

Fine-tuning improvement: 0.083



**Confusion matrix:**

```
              pred: analysis  hot_take  reaction
true analysis     [0]         [0]       [5]
true hot_take     [0]         [0]       [7]
true reaction     [0]         [0]       [18]
```

**Success bar (set in advance):**
1. Beats a lazy guesser — macro-F1 well above the majority baseline. → **❌ Not met.** An always-`reaction` guesser scores macro-F1 ≈ 0.25 on this test set, and the fine-tuned model also scored **0.25** — it collapsed into the lazy guesser. (The zero-shot baseline actually did better, at macro-F1 ≈ 0.40.)
2. Solid headline — macro-F1 ≥ ~0.70. → **❌ Not met.** Fine-tuned macro-F1 = **0.25** (accuracy 0.60, but carried entirely by the majority class).
3. Reliable across labels — no class below ~0.60 F1, with strong `analysis` recall and `hot_take` precision. → **❌ Not met.** `analysis` and `hot_take` both scored **0.00** F1 (zero recall — the model predicted `reaction` for every test post); only `reaction` cleared the bar (F1 0.75).

Accuracy rose (0.517 → 0.600), but the fine-tuned model learned to predict `reaction` for everything, which wins on accuracy only because `reaction` is 18 of 30 test posts. On the metric I committed to in advance (macro-F1), fine-tuning did not improve over the zero-shot baseline and failed all three success criteria. The likely cause is a small, imbalanced training set: with `analysis` and `hot_take` rare, collapsing to the majority class minimizes loss.

### Sample classifications


| Post snippet | True label | Predicted | Confidence | Correct? |
|---|---|---|---|---|
| "Which songs would play at your funeral" — a personal list of meaningful songs, no general claim | `reaction` | `reaction` | `[conf]` | ✅ |
| "Noisy Goth band Crippling Alcoholism… it took me a few listens but now I'm OBSESSED" | `reaction` | `reaction` | `[conf]` | ✅ |
| "Playboi Carti is deadass the best to ever do it… Whole Lotta Red is the best album" | `hot_take` | `reaction` | 0.38 | ❌ |
| "Songs that are ruined for you by something small" — sets up *why* small details break a song | `analysis` | `reaction` | 0.37 | ❌ |
| "What makes these three pop songs so good? (with AlbumOfTheYear scores)" | `analysis` | `reaction` | 0.38 | ❌ |

**Why the funeral-playlist prediction is reasonable:** the post is a personal list of songs the writer loves with no general claim about what makes music good which is exactly the definition of `reaction`.


## Error analysis

The 3 test posts the model still got wrong after fine-tuning — all misread **toward `reaction`** with low confidence (~0.37–0.38), highlighting that a casual or question framing and emotional tone pull borderline posts down the evidence ladder.

**#4 — "Playboi Carti" (true: `hot_take`, predicted: `reaction`, conf 0.38)**
The post is wrapped in highly emotional fan-voice ("deadass the best to ever do it," "the goat"), which surface-matches `reaction` even though it's actually a bold general claim about Carti's standing. The model latched onto the affective tone rather than the underlying unargued superlative — and the 0.38 confidence shows it was genuinely torn on the `hot_take`/`reaction` boundary.

**#6 — "Songs ruined by something small" (true: `analysis`, predicted: `reaction`, conf 0.37)**
This is a discussion prompt phrased in casual first-person ("I'm talking you fully get into the music, only to recoil…"), so it reads like a personal feeling rather than a structured argument. The `analysis` signal is abstract — it sets up an evaluative framework for *why* small details break a song — and lacks the named-evidence cues (dates, artists, citations) the model learned to tie to `analysis`.

**#11 — "Three pop songs" (true: `analysis`, predicted: `reaction`, conf 0.38)**
Although it cites concrete evidence (AlbumOfTheYear scores, specific tracks), the post leads with a bare question — "What makes these three songs so good to great?" — which pattern-matches a casual opinion prompt. The model weighted the question framing and short truncated body over the verifiable scoring evidence, missing the analytical intent.

**Takeaway:** the model under-weights *structural* argument cues (evaluative framing, cited scores) when a post is short or question-led, and over-weights emotional/casual tone. Likely fixes: more `analysis` examples that are question-led and more `hot_take` examples written in fan-voice.

## What the model captured vs. what I intended

I intended the model to learn whether a claim is backed by evidence (`analysis`), asserted without it (`hot_take`), or just a feeling (`reaction`). Instead it overfit to the class prior since `reaction` was ~60% of the data, it just predicted `reaction` for everything and ignored the evidence signal entirely. It learned the shape of my dataset but not the distinction my labels were about, and fixing that would need more balanced `analysis`/`hot_take` examples.

## Spec reflection

**How the spec guided me:** The spec's requirement to define evaluation metrics and a success bar before building forced me to commit to macro-F1 and per-class targets up front, rather than reaching for raw accuracy. 

**How my implementation diverged:** The spec targeted a balanced set (~65 per label), but in practice `analysis` posts were genuinely rare even after backfilling from `top`/most-discussed threads, so I accepted a somewhat imbalanced labeled set rather than padding `analysis` with borderline posts. Forcing balance would have meant relabeling weak `hot_take`s as `analysis`.Keeping the labels honest and leaning on macro-F1 (which already weights each class equally regardless of count) was the better trade and the error analysis confirms the cost showed up where expected: the model under-detects `analysis`.

## AI tool use

1. **Annotation assistance:** pre-labeled each post for my review and then I went in and checked each one manually and fixed the ones that didn't quite match up correctly
2. **Failure analysis:** grouped misclassifications into patterns, which I then verified against the actual posts and quantified before including here.
