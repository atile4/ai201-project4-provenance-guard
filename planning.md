### Detection Symbols

Signal 1: Holistic Check

- This signal checks the tone and wording of the text. AI tends to be more flat, vague, and generic when it writes, while human writing has more meaning and has a certain style.
- However, a blind spot comes from the fact that the system is essentially going off the vibe of the text, so it can't be quantitavely measured. This makes it difficult to verify that text is AI generated.
- Another blind spot comes from the way people can write; there are people who can write clean, formal, and fluent prose, but these are also qualities of AI-generated text.

Signal 2: Statistic Check

- This signal checks for the length and structure of the text, checking things like sentence-length variance, vocabulary diversity, and punctuation density. AI tends to have more uniform-length sentences and rhythm, while human writing can vary significantly more, being more irregular.
- A blind spot that comes from this is that naturally uniform writers could be flagged for AI, and smaller pieces of text would give more unreliable statistics.

If a false positive is found, instead of immediately flagging a user of AI-generated work, which would ruin our credibilty, it would instead use the "uncertain" label. The creator should appeal by showing proof.

The signal's output would be a score from 0 to 1, where higher means more AI. Since we have two signals, we can just combine them with a weighted average.

### Uncertainty representation

A confidence score of 0.6 means that the model's confidence is roughly in the middle between AI and not AI. We'll categorize the confidence levels as such:

Not AI: 0-0.40
Uncertain: 0.4 -- 0.7
Likely AI: 0.7 -- 1.0

As such, a score of 0.6 would mean that the model is uncertain.

### Transparency Label Design

High-confidence AI (score ≥ 0.70):

⚠️ Likely AI-generated. Our analysis suggests this content was probably created with AI assistance (confidence: high). This is an automated estimate, not a certainty. If you're the creator and disagree, you can appeal this label.

High-confidence human (score ≤ 0.40):

✓ Likely human-written. Our analysis found no strong signs of AI generation (confidence: high). This is an automated estimate and not a guarantee of authorship.

Uncertain (0.40 – 0.70):

❓ Inconclusive. Our analysis couldn't confidently determine whether this content was human- or AI-written. We're showing this honestly rather than guessing. No attribution claim is being made.

### Appeals workflow

The creator of the content can appeal.

They would provide their ID, and their reasoning as to why they think their label is wrong.

Three things: (1) flip that content's status from classified to under_review; (2) write an audit-log entry that records the appeal alongside the original decision — original verdict, original scores, plus the new reasoning and a timestamp; (3) return a confirmation that the appeal was received and is under review. No automated re-classification — a human handles it.

What a reviewer sees in the queue. Everything needed to make a fair call: the original text, the creator_id, the original verdict and combined confidence, both individual signal scores (so they can see which signal drove the call), the creator's stated reasoning, and the timestamps. The two separate signal scores matter here — a reviewer seeing "LLM said AI but stylometry said human" knows instantly it was a borderline disagreement case.

Some possible edge cases are non-native English speakers, where writers tend to write more formally or must use AI to be able to translate from native language to English, or shorter submissions, which gives the model too little data to work with.

## Architecture

```
SUBMISSION FLOW
   Creator
     |
     |  POST /submit  { text, creator_id }
     v
+--------------------+
|   /submit route    |
+--------------------+
     |
     |  raw text (sent to BOTH signals)
     +---------------------------+
     |                           |
     v                           v
+----------------+        +--------------------+
|  Signal 1      |        |  Signal 2          |
|  LLM / holistic|        |  Stylometric /     |
|  (Groq)        |        |  statistical       |
+----------------+        +--------------------+
     |                           |
     | signal_1 score (0-1)      | signal_2 score (0-1)
     +-------------+-------------+
                   |
                   v
        +-----------------------+
        |  Confidence Scorer    |
        |  (combine + calibrate)|
        +-----------------------+
                   |
                   | combined confidence score (0-1)
                   v
        +-----------------------+
        |   Label Generator     |
        |  (3 thresholds ->     |
        |   AI / uncertain /    |
        |   human)              |
        +-----------------------+
                   |
                   | label text + verdict
                   +-------------------+
                   |                   |
                   v                   v
        +-----------------+   +-------------------------+
        |   Audit Log     |   |   Response to creator   |
        | (scores, verdict|   | { content_id,           |
        |  signals, time) |   |   attribution,          |
        +-----------------+   |   confidence, label }   |
                              +-------------------------+

```

```
APPEAL FLOW
   Creator
     |
     |  POST /appeal  { content_id, creator_reasoning }
     v
+--------------------+
|   /appeal route    |
+--------------------+
     |
     |  content_id + reasoning
     v
+----------------------------+
|   Status Update            |
|   status -> "under review" |
+----------------------------+
     |
     |  appeal + original decision
     v
+----------------------------+
|   Audit Log                |
|  (logs appeal next to the  |
|   original classification) |
+----------------------------+
     |
     |  confirmation
     v
+----------------------------+
|   Response to creator      |
|  { status: under_review,   |
|    message }               |
+----------------------------+
```

Submission flow: When a creator submits text to POST /submit, the raw text is sent to both detection signals in parallel — the LLM (holistic) and the stylometric heuristics (statistical). Their two scores are merged by the confidence scorer into a single calibrated score, which maps to one of three transparency labels (likely AI, uncertain, likely human); the full decision is written to the audit log before the response returns to the creator.

Appeal flow: When a creator disputes a result via POST /appeal, the system updates that content's status to "under review," logs the appeal alongside the original classification in the audit log, and returns a confirmation — no automated re-classification occurs, leaving the final call to a human reviewer.

## AI Tool Plan

### M3 (submission endpoint + first signal):

I'll provide my AI tool the detection signals section (both descriptions for holistic and statistic).
I'll also ask it to generate a Flask app skeleton so I have an interface to work with. In addition, I'll ask it to write the first signal function, as well as tests for it to ensure it works before setting up the endpoint.

### M4 (second signal + confidence scoring):

I'll provide my AI tool my planned signal 2, the uncertainty diagram, and diagram. I'll also ask for a function that can statistically measure the text, and I'll check that scores vary between AI-generated and human text.

### M5 (production layer):

For this layer, I'll provide the diagram and label variants, and I'll ask for the label generation logic and the /appeal endpoint. To test, I'll use very AI generated text and historic human text, and compare output scores, as well as edge cases.
